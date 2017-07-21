---
author: dougbtv
comments: true
date: 2017-07-21 11:50:00-05:00
layout: post
slug: kube-custom-scheduler
title: Any time in your schedule? Try using a custom scheduler in Kubernetes
category: nfvpe
---

I've recently been interested in the idea of extending the scheduler in Kubernetes, there's a number of reasons why, but at the top of my list is looking at re-scheduling failed pods based on custom metrics -- specifically for high performance high availablity; like we need in telecom. In my search for learning more about it, I discovered the [Kube docs for configuring multiple schedulers](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/), and even better -- a practical application, a [toy scheduler](https://github.com/kelseyhightower/scheduler) created by the one-and-only-kube-hero [Kelsey Hightower](https://twitter.com/kelseyhightower). It's about a year old and Hightower is on his game, so he's using alpha functionality at time of authoring. In this article I modernize at least a component to get it to run in the contemporary day. Today our goal is to run through the toy scheduler and have it schedule a pod for us. We'll also dig into Kelsey's go code for the scheduler a little bit to get an intro to what he's doing.

Fire up your terminals, and let's get ready to schedule some pods -- with the NOT the default scheduler.

## Requirements

Simply have a Kubernetes 1.7 up and running for you. 1.6 might work, too. If you don't have  Kube running, may I suggest that you use my [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks, and follow my article about [installing a kube cluster on centos](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) (ignore that it says kube 1.5 -- same steps will produce a 1.7 cluster).

Also, I use an all-CentOS 7 lab environment, and while it might not be required, note that it colors the ancillary tools and viewpoint from which I create this tutorial.

We'll install a few deps, I wound up with a Go version 1.6.3, which appears to work fine, for your reference.

## Install our deps

I'm performing these steps on my kube master, feel free to run them where's appropriate for you. You'll need to install some packages, and you'll need to be able to use the `kubectl` utility in order to perform these.

Now, let's go and install the deps we need:

```
[centos@kube-master ~]$ sudo yum install -y git golang tmux
```

Now, make yourself a dir for your go source.

```
[centos@kube-master ~]$ mkdir -p gocode/src
```

## Clone and build the scheduler

Now let's clone up Hightower's code into there.

```
[centos@kube-master ~]$ cd gocode/src/
[centos@kube-master src]$ git clone https://github.com/kelseyhightower/scheduler.git
[centos@kube-master src]$ cd scheduler/
[centos@kube-master scheduler]$ pwd
/home/centos/gocode/src/scheduler
```

Alright now that we're there, first thing we'll do is build the annotator.

```
[centos@kube-master scheduler]$ cd annotator/
[centos@kube-master annotator]$ go build
[centos@kube-master annotator]$ ls annotator -lh
-rwxrwxr-x. 1 centos centos 7.8M Jul 21 15:23 annotator
```

Which will produce a binary for us.

Now, go and build the scheduler proper.

```
[centos@kube-master annotator]$ cd ../
[centos@kube-master scheduler]$ go build
[centos@kube-master scheduler]$ ls scheduler -lh
-rwxrwxr-x. 1 centos centos 7.7M Jul 21 15:24 scheduler
```

Go makes it easy, right!?

## Start your kubectl proxy

We need to run a `kubectl proxy`, which is a [HTTP proxy to access the kube API](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/) -- our scheduler here will rely on it.

Run tmux:

```
[centos@kube-master ~]$ tmux 
```

This will give you a new screen, in that screen run:

```
[centos@kube-master ~]$ kubectl proxy
```

You can exit this screen and let it keep running by hitting `ctrl+b` then `d`. To return to the screen execute `tmux a`.

## Run the annotation

Alright, we're going to create some "prices" for each of our nodes. The scheduler will use this and then start the pods on the node with the lowest price.

```
[centos@kube-master scheduler]$ cd annotator/
[centos@kube-master annotator]$ ./annotator 
kube-master 0.20
kube-minion-1 0.20
kube-minion-2 0.05
kube-minion-3 1.60
```

Each time you run the annotator, it'll generate new prices for you. If you just want to list the prices, list them like so:

```
[centos@kube-master annotator]$ ./annotator -l
kube-master 0.20
kube-minion-1 0.20
kube-minion-2 0.05
kube-minion-3 1.60
```

## Kick up a pod...

Alright, now create a resource definition yaml file with these contents:

```
[centos@kube-master scheduler]$ cat ~/nginx.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  template:
    metadata:
      #annotations:
      #  "scheduler.beta.kubernetes.io/name": hightower
      labels:
        app: nginx
      name: nginx
    spec:
      schedulerName: hightower
      containers:
        - name: nginx
          image: "nginx:1.11.1-alpine"
          resources:
            requests:
              cpu: "500m"
              memory: "128M"
```

Hightower had been using the annotation earlier, but, this is now core functionality so what I've done that's different is used the `schedulerName` property under the spec in the resource definition. As you can see it's `schedulerName: hightower` (and `hightower` is set as a constant as scheduler name in the go code, more on that later)

Now, let's create this pod:

```
[centos@kube-master annotator]$ kubectl create -f ~/nginx.yaml 
deployment "nginx" created
```

We can check out and see that this pod won't scheduler, which is what we want for now:

```
[centos@kube-master annotator]$ watch -n1 kubectl get pods
```

And you might wanna describe it, too...

```
[centos@kube-master annotator]$ watch -n1 kubectl describe pod nginx-881608959-gwnll
```

Cool, good it shouldn't have started yet.

## Start the scheduler

Feel free to run this in a tmux screen, but, I ran it in it's own window.

Fire it up!

```
[centos@kube-master scheduler]$ ./scheduler 
2017/07/21 15:32:36 Starting custom scheduler...
2017/07/21 15:32:38 Successfully assigned nginx-881608959-vk6t3 to kube-minion-2
```

Hurray! It scheduled it to `kube-minion-2` if you look at our pricing output, you'll see that is the lowest priced node when we generated prices. Run a `kubectl get pods` to double check and you can pick up the IP address with a `kubectl describe $the_pod_name` and curl it to your heart's content.

If you want, destroy the pod with a:

```
[centos@kube-master scheduler]$ kubectl delete -f ~/nginx.yaml 
```

And generate new prices with `./annotator/annotator` and run the scheduler again, and see it schedule it to another place when you `kubectl create -f` it.

## Let's inspect the toy scheduler go code.

So let's take a look at the code in [the toy scheduler](https://github.com/kelseyhightower/scheduler). This is really a gloss-over, but maybe can help point you (and later me!) in the right direction to figure out more about how to use these concepts to our own advantages.

The files we're interested in are:

* `main.go`: The main app which starts a couple handler goroutines
* `processor.go`: Where our goroutines live.
* `kubernetes.go`: The Kube API meat-and-potatoes
* `bestprice.go`: Our metric for scheduling.

(There's also the `./annotator/annotator.go`, which is a small util, feel free to poke at that too)

Generally, we have a `main.go` which is our handler, it starts up some [goroutines](https://www.golang-book.com/books/intro/10) that run two methods, both found in the `process.go` file:

* `monitorUnscheduledPods()` 
* `reconcileUnscheduledPods()`

These handle the goroutine logic (e.g. working with the wait group), perform a wait operation (I assume for polling for the rest of the logic), and then call the `schedulePod()` method also in `processor.go`.

The `monitorUnscheduledPods()` also calls the method `watchUnscheduledPods()` from `kubernetes.go` which is looking for those unscheduled pods for us (looks to be polling, but, there's some things named "event" which makes me wonder if it has a watch on those events, I'm unsure and I didn't dig further for now). The `watchUnscheduledPods()` method returns a channel to the pods it discovers. 

When there's a pod to be scheduled, finally a `bind()` method is called from `kubernetes.go` -- this calls the [binding core](https://kubernetes.io/docs/api-reference/v1.7/#binding-v1-core) in Kubernetes API, which can bind a pod to a node, for example.

The processor also looks at the `bestPrice()` method, which is in `bestprice.go` -- this look at the "prices" for each node and returns the lowest value price, this is how we determine which pod is going to go where.
