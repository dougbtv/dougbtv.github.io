---
author: dougbtv
comments: true
date: 2018-07-25 16:06:01-05:00
layout: post
slug: kubernetes-operator-tutorial
title: A Kubernetes Operator Tutorial? You got it, with Operator-SDK and an Asterisk Operator!
category: nfvpe
---

So you need a Kubernetes Operator Tutorial, right? I sure did when I start. So guess what? [I got that b-roll](https://www.youtube.com/watch?v=SItFvB0Upb8)!  In this tutorial, we're going to use the [Operator SDK](https://github.com/operator-framework/operator-sdk), and I definitely got myself up-and-running by following the [Operator Framework User Guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md). Once we have all that setup

## References

* [Helm Operator article from blog.openshift.com](https://blog.openshift.com/make-a-kubernetes-operator-in-15-minutes-with-helm/)
* [LinuxHint Operator tutorial](https://linuxhint.com/kubernetes_operator/)

## Requirements

* A CentOS 7 machine to use for development
    - These commands all reference CentOS, if you use Fedora (or something else), then it might take some conversion to get all the deps.
* Access to Kubernetes version 1.9 or later cluster
    - Need a tute for that? Check out my latest [Kubernetes install](http://dougbtv.com/nfvpe/2018/03/21/kubernetes-on-centos/) tutorial.
    - We will also cover a quick minikube installation
* Your favorite text editor.
* A [rubber duck for debugging](https://rubberduckdebugging.com/).

## Basic development environment setup

Alright, we've got some deps to work through. Including, ahem, [dep](https://golang.github.io/dep/docs/installation.html). I didn't include "root or your regular user" but in short, generally, just the `yum` & `systemctl` lines here require su, otherwise they should be your regular user (especially for setting up your `GOPATH` and installing dep).

Make sure you have git, and this is a good time to install whatever usual goodies you use.

```
$ yum install -y git
$ git config --global user.email "you@example.com"
$ git config --global user.name "Your Name"
```

Firstly, [install Docker](https://docs.docker.com/install/).

```
$ yum install -y yum-utils   device-mapper-persistent-data   lvm2
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ yum install docker-ce -y
$ systemctl enable docker
$ systemctl start docker
```

Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

$ yum install -y kubectl
```

Double check that you've got `bridge-nf-call-iptables` all good.

```
$ sudo /bin/bash -c 'echo "1" > /proc/sys/net/bridge/bridge-nf-call-iptables'
```

Install minikube (optional: if this is part of a cluster or otherwise have access to another cluster). I'm not generally a huge minikube fan, however, in this case we're working on a development environment (seeing that we're looking into building an operator), so it's actually appropriate here.

```
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.2/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
$ sudo /usr/local/bin/minikube start --vm-driver=none
```

If something went wrong and you need to restart minikube from scratch you can do so with:

```
$ sudo /usr/local/bin/minikube stop; cd /etc/kubernetes/; sudo rm -F *.conf; /usr/local/bin/minikube delete; cd -
```

Follow the instructions from minikube for setting up your `.kube` folder. I didn't have great luck with it, so I performed a `sudo su -` in order to run say, `kubectl get nodes` to see that the cluster was OK. In my case, this also meant that I had to bring the cluster up as root as well. 

Install a [nice-and-up-to-date-golang](http://go-repo.io).

```
$ rpm --import https://mirror.go-repo.io/centos/RPM-GPG-KEY-GO-REPO
$ curl -s https://mirror.go-repo.io/centos/go-repo.repo | tee /etc/yum.repos.d/go-repo.repo
$ yum install -y golang
```

I changed root's `~/.bash_profile` path (given my above Minikube situation) to:

```
export GOPATH=/home/centos/go
PATH=$PATH:$HOME/bin:$(go env GOPATH)/bin
export PATH
```


Setup your go environment a little, goal here being able to run binaries that are in your `GOPATH`'s `bin` directory.

```
$ mkdir -p ~/go/bin
$ export GOPATH=~/go
$ export PATH=$PATH:$(go env GOPATH)/bin
```

Ensure that directory exists...

```
mkdir -p /home/centos/go/bin
```

Install [dep](https://golang.github.io/dep/docs/installation.html).

```
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
```

Install the [operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md).

```
$ mkdir -p $GOPATH/src/github.com/operator-framework
$ cd $GOPATH/src/github.com/operator-framework
$ git clone https://github.com/operator-framework/operator-sdk
$ cd operator-sdk
$ git checkout master
$ export PATH=$PATH:$GOPATH/bin && make dep && make install
```

## Create your new project

We're going to create a sample project using the `operator-sdk` CLI tool. Note -- I used my own GitHub namespace here, feel free to replace it with yours. If not, cool, you can also get a halloween costume of me (and scare kids and neighbors!)

```
$ mkdir -p $GOPATH/src/github.com/dougbtv
$ cd $GOPATH/src/github.com/dougbtv
$ operator-sdk new hello-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached
$ cd hello-operator
```

Sidenote: For what it's worth, at some point I had tried a few versions of operator-sdk tools to try to fix another issue. During this, I had some complaint (when running `operator-sdk new ...`) that something didn't meet constraints (`No versions of k8s.io/gengo met constraints`), and it turned out it was this kind of [stale dep package cache](https://github.com/golang/dep/issues/699). So you can clear it as such:

```
[centos@operator-box github.com]$ rm -Rf $GOPATH/pkg/dep/sources
```

Also, ignore if it complains it can't complete the git actions, they're so simple you can just manage it as a git repo however you please.

## Inspecting the scaffolded project

Let's modify the types package to define what our CRD looks like...

Modify `./pkg/apis/cache/v1alpha1/types.go`, replace the two structs at the bottom like so:

```
type MemcachedSpec struct {
    // Size is the size of the memcached deployment
    Size int32 `json:"size"`
}
type MemcachedStatus struct {
    // Nodes are the names of the memcached pods
    Nodes []string `json:"nodes"`
}
```

And then update the generated code for the custom resources...

```
operator-sdk generate k8s
```

Then let's update the handler, it's @ `./pkg/stub/handler.go`

We'll replace that file in its entirety with [this example memcached deployment code from github](https://raw.githubusercontent.com/operator-framework/operator-sdk/master/example/memcached-operator/handler.go.tmpl). Just copy-pasta it, or curl it down, whatever you like.

You'll also need to change the github namespace in that file, replace it with your namespace + the project name you used during `operator-sdk new $name_here`. I changed mine like so:

```
$ sed -i -e 's|example-inc/memcached-operator|dougbtv/hello-operator|' pkg/stub/handler.go
```

Now, let's create the CRD. First, let's just `cat` (I'm a cat person, like, seriously I love cats, if you're a dog person you can stop reading this article right now, or, you probably use `less` as a pager too, dog people, seriously!) it and take a look...

```
$ cat deploy/crd.yaml
```

Now you can create it...

```
$ kubectl create -f deploy/crd.yaml
```

Once it has been created, you can see it's listed, but, there's no CRD objects yet...

```
$ sudo kubectl get memcacheds.cache.example.com
```

In the [Operator-SDK user guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md#build-and-run-the-operator) they list two options for running your SDK. Of course, the production way to do it is create a docker image and push it up to a registry, but... we haven't even compiled this yet, so let's go one step at a time and run in our local cluster.

```
$ operator-sdk up local
```

Cool, you'll see it initialize, and you might get an error you can ignore for now:

```
ERRO[0000] failed to initialize service object for operator metrics: OPERATOR_NAME must be set 
```

Alright, so what has it done? Ummm, nothing yet! Let's create a custom resource and we'll watch what it does... Create a custom resource yaml file like so:

```
$ cat deploy/my.crd.yaml 
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 3
```

Now let's apply it:

```
$ kubectl apply -f deploy/my.crd.yaml 
```

And we can go and watch what's happening here...

```
$ watch -n1 kubectl get deployment
```

You'll see that it's creating a bunch of memcached pods from a deployment! Hurray! Now we can modify that...

Let's edit the the `./deploy/my.crd.yaml` to have a `size: 4`, like so:

```
$ cat deploy/my.crd.yaml 
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 4
```

We can apply that, and then we'll take another look...

```
$ kubectl apply -f deploy/my.crd.yaml 
$ watch -n1 kubectl get deployment
```

Awesome, 4 instances going. Alright cool, we've got an operator running! So... Can we create our own?

# Creating our own operator!

Well, almost! What we're going to do now is use Doug's asterisk-operator.

## How the operator was created

Some of the things that I modified after I had the scaffold was..

* Updated the `types.go` to include the fields I needed.
* I moved the `/pkg/apis/cache/` to `/pkg/apis/voip/`
    - And changed references to `memcached` to `asterisk`
* Created a scheme to discover all IPs of the Asterisk pods
* Created REST API called to Asterisk to push the configuration

## Some things to check out in the code...

Aside from what we reviewed earlier when we were scaffolding the application -- which is argually the most interesting from a standpoint of "How do I create any operator that want?" The second most interesting, or, potentially most interesting if you're interested in Asterisk -- is how we handle the service discovery and dynamically pushing configuration to Asterisk.

You can find the bulk of this in the [handler.go](https://github.com/dougbtv/asterisk-operator/blob/master/pkg/stub/handler.go). Give it a skim through, and you'll find where it makes the actions of:

1. Creating the deployment and giving it a proper size based on the CRDs
2. How it figures out the IP addresses of each pod, and then goes through and uses those to cycle through all the instances and create SIP trunks to all of the other Asterisk instances.

But... What about making it better? This Operator is mostly provided as an example, and to "do a cool thing with Asterisk & Operators", so some of the things here are clearly in the proof-of-concept realm. A few of the things that it could use improvement with are...

1. It's not very graceful with how it handles waiting for the Asterisk instances to become ready. There's some timing issues with when the pod is created, and when the IP address is assigned. It's not the cleanest in that regard.
2. There's a complete "brute force" method by which it creates all the SIP trunks. If you start with say, 2 instances, and change to 3 instances -- well... It creates all of the SIP trunks all over again instead of just creating the couple new ones it needs, I went along with the idea of don't [prematurely optimize](http://wiki.c2.com/?PrematureOptimization). But, this could really justified to optimize it.

## What's the application doing?

![Asterisk Operator diagram](https://i.imgur.com/L4rvtiT.png)

In short the application really just does three things:

1. Watches a CRD to see how many Asterisk instances to create
2. Figures out the IP addresses of all the Asterisk instances, using the Kube API
3. Creates SIP trunks from each Asterisk instance to each other Asterisk instance, using [ARI push configuration](https://blogs.asterisk.org/2016/03/09/pushing-pjsip-configuration-with-ari/), allowing us to make calls from any Asterisk instance to any other Asterisk instance.

## Let's give the Asterisk Operator a spin!

This assumes that you've completed creating the development environment above, and have it all running -- you know, with golang and GOPATH all set, minikube running and the operator-sdk binaries available.



# Bonus: Helm Operator!

Let's follow the [15 minute operator with Helm tutorial](https://blog.openshift.com/make-a-kubernetes-operator-in-15-minutes-with-helm/). See how far we can get. This uses the [helm operator kit](https://github.com/operator-framework/helm-app-operator-kit).

Clone the operator kit, we'll use their example.

```
$ git clone https://github.com/operator-framework/helm-app-operator-kit.git
$ cd helm-app-operator-kit/
```

Now, build a Docker image. Note: You'll probably want to change the name (from `-t dougbtv/...` to your name, or someone else's name if that's how you roll).

```
docker build \
  --build-arg HELM_CHART=https://storage.googleapis.com/kubernetes-charts/tomcat-0.1.0.tgz \
  --build-arg API_VERSION=apache.org/v1alpha1 \
  --build-arg KIND=Tomcat \
  -t dougbtv/tomcat-operator:latest .
```

Docker login and then push the image.

```
$ docker login
$ docker push dougbtv/tomcat-operator:latest
```

Alright, now there's a series of things we've got to customize. There's more instructions on [what needs to be customized](https://github.com/operator-framework/helm-app-operator-kit#instructions), too, if you need it.

```
# this can stay changed to "tomcat"
$ sed -i -e 's/<chart>/tomcat/' helm-app-operator/deploy/operator.yaml 

# this you should change to your docker namespace
$ sed -i -e 's|quay.io/<namespace>|dougbtv|' helm-app-operator/deploy/operator.yaml

# Change the group & kind to match what we had in the docker build.
$ sed -i -e 's/group: example.com/group: apache.org/' helm-app-operator/deploy/crd.yaml 
$ sed -i -e 's/kind: ExampleApp/kind: Tomcat/' helm-app-operator/deploy/crd.yaml 

# And the name has to match that, too
$ sed -i -e 's/name: exampleapps.example.com/name: exampleapps.apache.org/' helm-app-operator/deploy/crd.yaml

# Finally update the Custom Resource to be what we like.
$ sed -i -e 's|apiVersion: example.com/v1alpha1|apiVersion: apache.org/v1alpha1|' helm-app-operator/deploy/cr.yaml
$ sed -i -e 's/kind: ExampleApp/kind: Tomcat/' helm-app-operator/deploy/cr.yaml
```

Now let's deploy all that stuff we created!

```
$ kubectl create -f helm-app-operator/deploy/crd.yaml
$ kubectl create -n default -f helm-app-operator/deploy/rbac.yaml
$ kubectl create -n default -f helm-app-operator/deploy/operator.yaml
$ kubectl create -n default -f helm-app-operator/deploy/cr.yaml
```

