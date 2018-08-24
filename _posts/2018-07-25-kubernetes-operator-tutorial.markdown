---
author: dougbtv
comments: true
date: 2018-07-25 16:06:01-05:00
layout: post
slug: kubernetes-operator-tutorial
title: A Kubernetes Operator Tutorial? I got that b-roll.
category: nfvpe
---

So you need a Kubernetes Operator Tutorial, right? I sure did when I start. So guess what? [I got that b-roll](https://www.youtube.com/watch?v=SItFvB0Upb8)! 

In this tutorial, we're going to use the [Operator SDK](https://github.com/operator-framework/operator-sdk), and I definitely got myself up-and-running by following the [Operator Framework User Guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md).

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

Install minikube (optional: if this is part of a cluster or otherwise have access to another cluster). I'm not generally a huge minikube fan, however, in this case we're working on a development environment (seeing that we're looking into building an operator), so it's actually appropriate here.

```
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.2/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
$ sudo /usr/local/bin/minikube start --vm-driver=none
```

If something went wrong and you need to restart minikube from scratch you can do so with:

```
$ sudo /usr/local/bin/minikube stop; cd /etc/kubernetes/; sudo rm -F *.conf; /usr/local/bin/minikube delete; cd -
```

Follow the instructions from minikube for setting up your `.kube` folder. I didn't have great luck with it, so I performed a `sudo su -` in order to run say, `kubectl get nodes` to see that the cluster was OK. In my case, this also meant that I had to bring the cluster up as root as well. I changed root's `~/.bash_profile` path to:

```
export GOPATH=/home/centos/go
PATH=$PATH:$HOME/bin:$(go env GOPATH)/bin
export PATH
```

Install a [nice-and-up-to-date-golang](http://go-repo.io/).

```
$ rpm --import https://mirror.go-repo.io/centos/RPM-GPG-KEY-GO-REPO
$ curl -s https://mirror.go-repo.io/centos/go-repo.repo | tee /etc/yum.repos.d/go-repo.repo
$ yum install golang
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
```

Setup your go environment a little, goal here being able to run binaries that are in your `GOPATH`'s `bin` directory.

```
$ mkdir -p ~/go/bin
$ export GOPATH=~/go
$ export PATH=$PATH:$(go env GOPATH)/bin
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
$ make dep && make install
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

You'll also need to change the github namespace in that file, replace it with your namespace + the project name you used during `operator-sdk new $name_here`. I changed mine like so:

```
$ sed -i -e 's|example-inc/memcached-operator|dougbtv/hello-operator|' pkg/stub/handler.go
```

We'll replace that file in its entirety with [this example memcached deployment code from github](https://raw.githubusercontent.com/operator-framework/operator-sdk/master/example/memcached-operator/handler.go.tmpl). Just copy-pasta it, or curl it down, whatever you like.

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
$ sudo kubectl create -f deploy/crd.yaml
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

# Helm Operator!