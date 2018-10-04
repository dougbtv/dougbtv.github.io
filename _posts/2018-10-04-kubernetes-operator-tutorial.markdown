---
author: dougbtv
comments: true
date: 2018-10-04 11:55:01-05:00
layout: post
slug: kubernetes-operator-tutorial
title: A Kubernetes Operator Tutorial? You got it, with the Operator-SDK and an Asterisk Operator!
category: nfvpe
---

So you need a Kubernetes Operator Tutorial, right? I sure did when I started. So guess what? [I got that b-roll](https://www.youtube.com/watch?v=SItFvB0Upb8)!  In this tutorial, we're going to use the [Operator SDK](https://github.com/operator-framework/operator-sdk), and I definitely got myself up-and-running by following the [Operator Framework User Guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md). Once we have all that setup -- oh yeah! We're going to run a custom Operator. One that's designed for Asterisk, it can spin up Asterisk instances, discover them as services and dynamically create SIP trunks between n-number-of-instances of Asterisk so they can all reach one another to make calls between them. Fire up your terminals, it's time to get moving with Operators.

What exactly are Kubernetes Operators? In my own description -- Operators are applications that manage other applications, specifically with tight integration with the Kubernetes API. They allow you build in your own "operational knowledge" into them, and perform automated actions when managing those applications. You might also want to see what [CoreOS has to say on the topic](https://coreos.com/operators/), read their [blog article where they introduced operators](https://coreos.com/blog/introducing-operators.html).

Sidenote: Man, what an overloaded term, Operators! In the telephony world, well, we have operators, like... a [switchboard operator](https://www.google.com/search?tbm=isch&source=hp&biw=1916&bih=926&ei=9Eq2W8XkGtC05gKu74Uw&q=switchboard+operator&oq=switchboard+operator&gs_l=img.3..0l10.505.2736..2811...0.0..1.103.1267.18j1......2....1..gws-wiz-img.....0..35i39.XYtoYQsWkvs) (I guess that one's at least a little obsolete). Then we have platform operators, like... sysops. And we have how things operate, and the operations they perform... Oh my.

A guy on my team said (paraphrased): "Well if they're applications that manage applications, then... Why write them in Go? Why not just write them in bash?". He was... Likely kidding. However, it always kind of stuck with me and got me to think about it a lot. One of the main reasons why you'll see these written in Go is because it's going to be the default choice for interacting with the Kubernetes API. There's likely other ways to do it -- but, all of the popular tools for interacting with it are written in Go, just like Kubernetes itself. The thing here is -- you probably care about managing your application running in Kubernetes with an operator because you care about integrating with the Kubernetes API.

One more thing to keep in mind here as we continue along -- the idea of CRDS -- Custom Resource Definitions. These are the lingua franca of Kubernetes Operators. We often watch what these are doing and take actions based on them. What's a CRD? It's often described as "a way to extend the Kubernetes API", which is true. The thing is -- that sounds SO BIG. It sounds daunting. It's not really. CRDs, in the end, are just a way for you to store some of your own custom data, and then access it through the Kubernetes API. Think of it as some meta data you can push into the Kube API and then access it -- so if you're interacting with the Kube API, it's simple to store some of your own data, without having to roll-your-own way of otherwise storing it (and otherwise reading & writing that data).

Today we have a big agenda for this blog article... Here's what we're going to do:

* Create a development environment where we can use the [operator-sdk](https://github.com/operator-framework/operator-sdk)
* Create own application as scaffolded by the Operator SDK itself.
* Spin up the [asterisk-operator](https://github.com/dougbtv/asterisk-operator), dissect it a little bit, and then we'll run it and see it in action.
* Lastly, we'll introduce the Helm Operator, a way to kind of lower the barrier of entry that allows you to create a Kubernetes Operator using Helm, and it might solve some of your problems that you'd use an Operator for without having to slang any golang.

## References

Here's a few articles that I used when I was building this article myself.

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

Alright, we've got some deps to work through. Including, ahem, [dep](https://golang.github.io/dep/docs/installation.html). I didn't include "root or your regular user" but in short, generally, just the `yum` & `systemctl` lines here require su, otherwise they should be your regular user.

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

It'll take a few minutes while it downloads a few container images from which it runs Kubernetes.

If something went wrong and you need to restart minikube from scratch you can do so with:

```
$ sudo /usr/local/bin/minikube stop; cd /etc/kubernetes/; sudo rm -F *.conf; /usr/local/bin/minikube delete; cd -
```

Follow the instructions from minikube for setting up your `.kube` folder. I didn't have great luck with it, so I performed a `sudo su -` in order to run say, `kubectl get nodes` to see that the cluster was OK. In my case, this also meant that I had to bring the cluster up as root as well. 

You can test that your minikube is operational with:

```
kubectl get nodes
```

It should list just a single node.

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

If you do the same thing you might want to be mindful of the `/home/user` in that path.

Setup your go environment a little, goal here being able to run binaries that are in your `GOPATH`'s `bin` directory.

```
$ mkdir -p ~/go/bin
$ export GOPATH=~/go
$ export PATH=$PATH:$(go env GOPATH)/bin
```

Ensure that directory exists...

```
mkdir -p $GOPATH/bin
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

Modify `./pkg/apis/cache/v1alpha1/types.go`, replace the two structs at the bottom (that say `// Fill me`) like so:

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
$ kubectl get memcacheds.cache.example.com
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

Well, almost! What we're going to do now is use Doug's asterisk-operator. Hopefully there's some portions here that you can use as a springboard for your own Operator.

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

First things first -- make sure you pull the image we'll use in advance, this will make for a lot less confusing waiting when you first start the operator itself.

```
docker pull dougbtv/asterisk-example-operator
```

Then, clone the asterisk-operator git repo:

```
mkdir -p $GOPATH/src/github.com/dougbtv && cd $GOPATH/src/github.com/dougbtv
git clone https://github.com/dougbtv/asterisk-operator.git && cd asterisk-operator
```

We'll need to create the CRD for it:

```
kubectl create -f deploy/crd.yaml
```

Next... We'll just start the operator itself!

```
operator-sdk up local
```

Ok, cool, now, we'll create a CRD so that the operator sees it and spins up asterisk instances -- open up a new terminal window for this.

```
cat <<EOF | kubectl apply -f -
apiVersion: "voip.example.com/v1alpha1"
kind: "Asterisk"
metadata:
  name: "example-asterisk"
spec:
  size: 2
  config: "an unused field."
EOF
```

Take a look at the output from the operator -- you'll see it logging a number of things. It has some waits to properly wait for Asterisk's IP to be found, and for Asterisk instances to be booted -- and then it'll log that it's creating some trunks for us.

Check out the deployment to see that all of the instances are up:

```
watch -n1 kubectl get deployment
```

You should see that it desires to have 2 instances, and that it's fulfilled those instances. It does this as it has created a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Let's go ahead and exec into one of the Asterisk pods, and we'll run the Asterisk console...

```
kubectl exec -it $(kubectl get pods -o wide | grep asterisk | head -n1 | awk '{print $1}') -- asterisk -rvvv
```

Let's show the AORs (addresses of record):

```
example-asterisk-6c6dff544-2wfwg*CLI> pjsip show aors

      Aor:  <Aor..............................................>  <MaxContact>
    Contact:  <Aor/ContactUri............................> <Hash....> <Status> <RTT(ms)..>
==========================================================================================

      Aor:  example-asterisk-6c6dff544-wnkpx                     0
    Contact:  example-asterisk-6c6dff544-wnkpx/sip:anyuser 1a830a6772 Unknown         nan
```

Ok, cool, this has a trunk setup for us, the trunk name in the `Aor` field is `example-asterisk-6c6dff544-wnkpx`. Go ahead and copy that value in your own terminal (yours will be different, if it's not different -- leave your keyboard right now, and go buy a lotto ticket).

We can use that to originate a call, I do so with:

```
example-asterisk-6c6dff544-2wfwg*CLI> channel originate PJSIP/333@example-asterisk-6c6dff544-wnkpx application wait 2
    -- Called 333@example-asterisk-6c6dff544-wnkpx
    -- PJSIP/example-asterisk-6c6dff544-wnkpx-00000000 answered
```

And we can see that there's a call that's been originated, and it has been answered by the other end! Go ahead an `quit` for now.

Ok -- but, here comes the cool stuff. Let's increase the size of our cluster, we requested 2 instances of Asterisk earlier, now we'll bump it up to 3.

```
cat <<EOF | kubectl apply -f -
apiVersion: "voip.example.com/v1alpha1"
kind: "Asterisk"
metadata:
  name: "example-asterisk"
spec:
  size: 3
  config: "an unused field."
EOF
```

Now our `kubectl get deployment` will show us that we have three, but! Better yet, we have all the SIP trunks created for us. Let's exec in and look at the AORs again.

```
kubectl exec -it $(kubectl get pods -o wide | grep asterisk | head -n1 | awk '{print $1}') -- asterisk -rvvv
```

Then we'll do the same and show the AORs:

```
example-asterisk-6c6dff544-2wfwg*CLI> pjsip show aors

      Aor:  <Aor..............................................>  <MaxContact>
    Contact:  <Aor/ContactUri............................> <Hash....> <Status> <RTT(ms)..>
==========================================================================================

      Aor:  example-asterisk-6c6dff544-k2m7z                     0
    Contact:  example-asterisk-6c6dff544-k2m7z/sip:anyuser 0d391d57b2 Unknown         nan

      Aor:  example-asterisk-6c6dff544-wnkpx                     0
    Contact:  example-asterisk-6c6dff544-wnkpx/sip:anyuser 1a830a6772 Unknown         nan
```

Ah ha! Now there's 2 trunks available, the operator went and created a new one for us to the new Asterisk instance.

And we can originate a call to it, too!

```
example-asterisk-6c6dff544-2wfwg*CLI> channel originate PJSIP/333@example-asterisk-6c6dff544-wnkpx application wait 2
    -- Called 333@example-asterisk-6c6dff544-wnkpx
    -- PJSIP/example-asterisk-6c6dff544-wnkpx-00000001 answered
```

And there you have it -- you can do it for n-number of instances. I tested it out with 33 instances, which works out to 1056 trunks (counting both sides) and... While it took like 15ish minutes, which felt like forever... It takes me longer than that to create 2 or 3 by hand! So... Not a terrible trade off.

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

