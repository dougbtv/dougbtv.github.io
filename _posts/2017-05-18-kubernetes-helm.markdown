---
author: dougbtv
comments: true
date: 2017-05-18 16:30:01-05:00
layout: post
slug: kubernetes-helm
title: Sailing the 7 seas with Kubernetes Helm
category: nfvpe
---

Helm is THE Kubernetes Package Manager. Think of it like a way to `yum` install your applications, but, in Kubernetes. You might ask, "Heck, a deployment will give me most of what I need. Why don't I just create a pod spec?" The thing is that Helm will give you a way to template those specs, and give you a workflow for managing those templates by what it calls "charts". Today we'll go ahead and step through the process of installing Helm, installing a pre-built pod given an existing chart, and then we'll make an existing chart of our own and deploy it -- and we'll change some template values so that it runs in a couple different ways. Ready? Anchors aweigh...

The beginning base of these instruction on the [official quickstart guide](https://github.com/kubernetes/helm/blob/master/docs/quickstart.md). We'll then extrapolate from there as there's a few considerations that we have now.

1. We need to configure RBAC, which isn't officially covered yet. The official way they say to do it is to turn off RBAC -- nope, not going to do that.
2. We're also going to make our own Helm charts, which aren't covered in the quick start guide, so we'll expand from there.

## Requirements

So, this assumes you've already got a Kubernetes cluster up and running, and usually... These articles assume CentOS 7.3 running. It might not exactly require CentOS 7.3 this time, but, just know that's my reference, and I'm using Kubernetes 1.6. 

If you don't have a Kubernetes cluster up, may I recommend using my [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks -- and I've got an article [detailing how to use those playbooks](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/).

Optionally -- you can create persistent volumes. You can skip this step if you want, but, the example charts that we will install require some volume persistence. We'll run it with persistence turned off, but, it's "more realistic" if-you-will. And if you don't have persistent volumes setup, you might want to try [my method for using GlusterFS to back persistent volumes, as detailed in this blog post](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/).

## About Helm

Helm is really two parts, a client and a server. The client is `helm` and the server is `tiller` -- all the boat references! Cause the definition of your applications are called `charts`.

So if you're at the helm of a ship, and you steer (according to your charts), you'd move your tiller. See? All the ships!

These charts are essentially templates for how to deploy your pods. Without helm, you'd just create specs which are yaml files which define how the pod is to be run. But, using helm -- we can make charts which make for more flexible specs. That way we can run the same application with differing parameters in the same or a different cluster.

Why not template them with Ansible, then? You could, too. But, using helm gives us a more direct work-flow for define the charts and deploying them, and should free up our playbooks to allow for lower-level infrastructure creation, and let our applications be abstracted from that, and let us leverage what Kubernetes has to offer without having to overly complicate our playbooks for applications -- which should likely require more frequent reconfiguration than the underlying pieces. For the record, in my opinion -- using Ansible isn't the wrong way. It's just another way.

## Download Helm

Let's pick out a version from the [github releases of helm](https://github.com/kubernetes/helm/releases) and download the binary onto our Kubernetes master server.

```
[centos@kube-master ~]$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.4.1-linux-amd64.tar.gz > helm.tar.gz
[centos@kube-master ~]$ tar -xzvf helm.tar.gz 
[centos@kube-master ~]$ chmod +x linux-amd64/helm 
[centos@kube-master ~]$ sudo cp linux-amd64/helm /usr/local/bin
```

Now, let's check its version.

```
[centos@kube-master ~]$ helm version
Client: &version.Version{SemVer:"v2.4.1", GitCommit:"46d9ea82e2c925186e1fc620a8320ce1314cbb02", GitTreeState:"clean"}
```

It will also take a second to complete, and then timeout and probably complain that it can't connect to tiller. Which is fine for now. So, that's coming up soon.

## Run `helm init`

```
[centos@kube-master ~]$ helm init
```

That should start tiller for us -- so you'll have to watch for it come up, go ahead and `watch -n1 kubectl get pods`

And we'll have to create an [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) for it, too. I used [this gist](https://gist.github.com/mgoodness/bd887830cd5d483446cc4cd3cb7db09d) as a reference.

```
[centos@kube-master ~]$ kubectl -n kube-system create sa tiller
serviceaccount "tiller" created
[centos@kube-master ~]$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding "tiller" created
[centos@kube-master ~]$ kubectl -n kube-system patch deploy/tiller-deploy -p '{"spec": {"template": {"spec": {"serviceAccountName": "tiller"}}}}'
deployment "tiller-deploy" patched
```

Go ahead and watch your pods, cause it's going to restart the tiller pod, so do something like `watch -n1 kubectl get pods --all-namespaces` until it comes back.

## Let's run an example app

Let's go ahead and update our repo.

    [centos@kube-master ~]$ helm repo update

And then we can install say... MongoDB (I'm wearing a MongoDB t-shirt today, so why not that one). If you'd like to install something else [checkout the official "stable" repo and see what's available](https://github.com/kubernetes/charts/tree/master/stable).

```
[centos@kube-master ~]$ helm install --set persistence.enabled=false stable/mongodb
```

Note that we're already doing something that sets Helm apart from "just using a spec file". Like, if you're familiar with my other tutorials you may have seen me create pods from specs before, where I've created a yaml file, and then I tell kubernetes to create it with something like `kubectl create -f mongodb.yaml`.

So running that `helm install` is going to give you some output like this (I clipped out some of the output)...

```
[centos@kube-master ~]$ helm install --set persistence.enabled=false stable/mongodb
[...snip...]
NOTES:
MongoDB can be accessed via port 27017 on the following DNS name from within your cluster:
silly-ladybird-mongodb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run silly-ladybird-mongodb-client --rm --tty -i --image bitnami/mongodb --command -- mongo --host silly-ladybird-mongodb

```

So let's go ahead and use mongo for fun. Note, this command is going to take a while because Kube is going to pull a new image for you.

```
[centos@kube-master ~]$ kubectl run silly-ladybird-mongodb-client --rm --tty -i --image bitnami/mongodb --command -- mongo --host fallacious-giraffe-mongodb
```

You might have to hit enter, and it lets you know to do that too.

Let's do something with it while we're here.

```
> use kitchen;
switched to db kitchen
> db.kitchen.insert({"beer": {"heady topper": 4,"sip of sunshine": "awww yeah"}})
> db.kitchen.find().pretty()
{
  "_id" : ObjectId("591cb31956ed4d11bd5b82c0"),
  "beer" : {
    "heady topper" : 4,
    "sip of sunshine" : "awww yeah"
  }
}

```

Ok, cool, it works!

So what about some visibility of what charts we have deployed? Run `helm list` to check it out for yourself. This is a list of what are referred to as "releases".

```
[centos@kube-master ~]$ helm list
NAME        REVISION  UPDATED                   STATUS    CHART           NAMESPACE
aged-uakari 1         Wed May 17 20:26:18 2017  DEPLOYED  mongodb-0.4.10  default  
```

Then you can go ahead and remove this sample one.

```
[centos@kube-master ~]$ helm delete aged-uakari
release "aged-uakari" deleted
```


## Let's create our own chart

I got a little help for creating a first chart [from this blog post](https://deis.com/blog/2016/getting-started-authoring-helm-charts/). Let's go ahead and create our own.

We're going to try to create an nginx instance that serves a photograph of a pickle. Because, that is absurd enough for me.

