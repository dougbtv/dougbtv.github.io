---
author: dougbtv
comments: true
date: 2017-02-22 11:35:01-05:00
layout: post
slug: multus-cni
title: So you want to expose a pod to multiple network interfaces? Enter Multus-CNI
category: nfvpe
---

Sometimes, one isn't enough. Especially when you've got network requirements that aren't just "your plain old HTTP API". By default in Kubernetes, a pod is exposed only to a loopback and a single interface as assigned by your pod networking. In the telephony world, something we love to do is isolate our signalling, media, and management networks. If you've got those in separate NICs on your container host, how do you expose them to a Kubernetes pod? Let's plug in the [CNI](https://github.com/containernetworking/cni) (container network interface) plugin called [multus-cni](https://github.com/Intel-Corp/multus-cni) into our Kubernetes cluster and we'll expose multiple network interfaces to a (very simple) pod.

Our goal here is going to be to spin up a pod using [the technique describe in this article](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) I wrote about spinning up Kubernetes 1.5 on CentOS -- from there, we'll install multus-cni and configure pod networking so that we expose a pod to two interfaces: 1. To Flannel, and 2. To the host's eth0 nic. 

We'll cover two methods here -- the first being to use my [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks and spin it up with "no CNI networking" and install CNI by hand -- this will allow us to familiarize ourselves with the components in detail here. Later, a secondary method using those playbooks will be introduced where it automatically sets up multus-cni using the playbooks, too.

If you're itching to get to the "how to" skip down to the "Let's get started" section below. 

You'll notice that I refer to multus-cni interchangably through this article as "multus-cni" (the Git clone's name) or "Multus", which I guess I inferred from their documentation which reads "MULTUS CNI Plugin", and then describes that "Multus" is Latin -- and I looked it up myself and it generally translates to "many" or "numerous", and their documentation tends to hint at that it may be the root of the prefix "multi-" -- so I checked the etymology on [Merriam-Webster](https://www.merriam-webster.com/dictionary/multi-) and they're right -- it is indeed!

## Taking a look at CNI

Through this process, I got to get exposed to a number of different pieces of CNI that became more and more valuable through the process. And maybe you'll want to learn some more about CNI, too. I won't belabor what CNI is here, but, quickly... 

One of the first thing is that CNI is not [libnetwork](https://github.com/docker/libnetwork) (the default way Docker connects containers), and you might be wondering [why does k8s doesn't use libnetwork](http://blog.kubernetes.io/2016/01/why-Kubernetes-doesnt-use-libnetwork.html). And if you want to hear it straight from the horse's mouth, check out [the CNI specifications](https://github.com/containernetworking/cni/blob/master/SPEC.md).

But the most concise way to describe CNI is (quoted from the spec):

> [CNI is] a generic plugin-based networking solution for application containers on Linux

## So, what's multus-cni?

That being said multus-cni is a plugin for CNI -- one that allows a pod to be connected to multiple network interfaces, or as the GitHub project description reads rather succinctly, it's "Multi-homed pod cni". And basically what we're going to do is build some Go code for it, and then go ahead and put the binary in the right place so that CNI can execute it. It's... That easy!

Almost.

Multus is actually fairly simple, but, it requires that you understand some other portions of CNI. One of the most important places you'll need to go is the documentation for [the included CNI plugins](https://github.com/containernetworking/cni/tree/master/Documentation). Because, in my own words as a user -- basically Multus is a wrapper for combining other CNI plugins and basically lets you define a list of plugins you're going to use to expose multiple interfaces to a pod.

I struggled at first especially because I didn't exactly grok that. I was trying to modify a configuration that I thought was specific to multus-cni, but, I was missing that it was wrapping the configuration for other CNI plugins.

Luckily for me, I picked up a little community help along the way and got the last few pieces sorted out. Kuralamudhan gave me some input here in [this GitHub issue](https://github.com/Intel-Corp/multus-cni/issues/3), and he was very friendly about offering some assistance. Additionally, in the [Kubernetes slack](https://kubernetes.slack.com), [Yaron Haviv](https://twitter.com/yaronhaviv) shared his sample configuration. Between "my own way" (which you'll see in a little bit), Kuralamudhan pointing out that sections of the config are related to other plugins, and having a spare reference from Yaron I was able to get Multus firing on all pistons. 

## Requirements for this walk-through

The technique used here is based on [the technique used in this article](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) to spin up k8s 1.5 on CentOS. And this technique leverages my [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks available on GitHub. By default it spins up 3 virtual machines on a virtual machine host. You can bring your own virtual machines (or bare metal machines) just make sure they're (generally the latest) CentOS 7 installs. That article may familiarize you with the structure of the playbooks -- especially if you need some more detail on bringing your own inventory.

Also, this uses Ansible playbooks, so you'll need an ansible machine. 

Note that I'm going to skip over some of the details of how to customize the Ansible inventories, so, refer to the previous article if you're lost.

## Let's get started

Alright, let's pull the rip cord! Ok, first thing's first clone my kube-centos-ansible repo. 

```
$ git clone https://github.com/dougbtv/kube-centos-ansible.git
$ cd kube-centos-ansible
```

Go ahead and modify `./inventory/virt-host.inventory` to suit your virtual host, and let's spin up some virtual machines.

```
$ ansible-playbook -i inventory/virthost.inventory virt-host-setup.yml
```

Based on the results of that playbook modify `./inventory/vms.inventory`. Now that it's up, we're going to run the `kube-install.yml` -- but, with a twist. By default this playbook uses Flannel only. So we're going to pass in a variable that says to the playbook "skip setting up any CNI plugins". So we'll run it like below.

Note: This kicks off the playbooks in a way that allows us to manually configure multus-cni so we can inspect it. If you'd like to let the playbook do all the install for you, you can -- skip down to the section near the bottom titled: 'Welcome to "easy mode"'.

```
$ ansible-playbook -i inventory/vms.inventory kube-install.yml --extra-vars "pod_network_type=none"
```

If you ssh into the master, you should be able to `kubectl get nodes` and see a master and two nodes at this point.

Now, we need to compile and install multus-cni, so let's run that playbook. It runs on all the VMs.

```
$ ansible-playbook -i inventory/vms.inventory multus-cni.yml
```

Basically all this playbook does is install the dependencies (like, golang and git) and then clones the multus-cni repo and builds the go binaries. It then copies those binaries into `/opt/cni/bin/` so that CNI can run the plugins from there.

## Inspecting the multus-cni configuration

Alright, so now let's ssh into the master, and we'll get a config downloaded here and take a look. 

We're going to use [this yaml file I've posed as a gist](https://gist.github.com/dougbtv/cf05026e48e5b8aa9068a7f6fcf91a56)

Let's curl that down to the master, and then we'll take a look at a few parts of it.

```
[centos@kube-master ~]$ curl https://gist.githubusercontent.com/dougbtv/cf05026e48e5b8aa9068a7f6fcf91a56/raw/dd3dfbf5e440abea8781e27450bb64c31e280857/multus-working.yaml > multus.yaml
```

Generally, my idea was to take the [Flannel pod networking yaml](https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml) and modify it to suit multus-cni, seeing they play together. In fact, I couldn't get it to work with just a multus-cni config alone. If you compare and contrast the two, you'll notice the Flannel yaml (say that outloud three times in a row) has been borrowed from heavily.

Go ahead and `cat` the `multus.yaml` file so we can look at it, or bring it up in your favorite editor as long as it's not emacs. If it is indeed emacs, the next step in this walk-through is for you to go jump in a lake and think about your life for a little while ;) (JK, I love you emacs brethren, you're just... weird.)

The Multus configuration is JSON packed inside a yaml configuration for Flannel, generally. According to the CNI spec "The network configuration is in JSON format and can easily be stored in a file". We're defining a k8s [ConfigMap](https://kubernetes.io/docs/user-guide/configmap/) which has the Multus configuration within. You see, Multus works in concert with other CNI plugins. 

First up, [looking at lines 17-45](https://gist.github.com/dougbtv/cf05026e48e5b8aa9068a7f6fcf91a56#file-multus-working-yaml-L17-L45) this is the Multus configuration proper. 

Note there's a JSON list in here called `delegates` and is a list of two items.

The top-most element is `"type": "macvlan"` and uses the [CNI plugin macvlan](https://github.com/containernetworking/cni/blob/master/Documentation/macvlan.md). This is what we use to map a bridge to `eth0` in the virtual machine. We also then specify a network range which is that of the default libvirt br0. 

The second element is the one that's specific to multus-cni, and is `"type": "flannel"` but has an element specific to multus-cni, which is `"masterplugin": true`. 

Continuing through the file, we'll later see a `DaemonSet` defined which has the pods for Flannel's networking. Later on in that file, we'll see that there's a [command which is run on line 95](https://gist.github.com/dougbtv/cf05026e48e5b8aa9068a7f6fcf91a56#file-multus-working-yaml-L95). This basically takes the JSON from the ConfigMap and copies it to the proper place on the host.

What's great about this step is that the config winds up in the right place on each machine in the cluster. Without this step, I'm not sure how we'd get that configuration properly setup on each machine without something like Ansible to put the config where it should be.

## Applying the multus configuration

Great, so you've already downloaded the `multus.yaml` file onto the master, let's go ahead and apply it.

```
[centos@kube-master ~]$ kubectl apply -f multus.yaml 
serviceaccount "multus" created
configmap "kube-multus-cfg" created
daemonset "kube-multus-ds" created
```

Let's watch the pods, and wait until each instance of the pod is running on each node in the cluster.

```
[centos@kube-master ~]$ watch -n1 kubectl get pods --all-namespaces
```

In theory you should have three lines which looks about like this when they're ready.

```
[centos@kube-master ~]$ kubectl get pods --all-namespaces | grep multus
kube-system   kube-multus-ds-cgtr6                  2/2       Running   0          1m
kube-system   kube-multus-ds-qq2tm                  2/2       Running   0          1m
kube-system   kube-multus-ds-vkg3r                  2/2       Running   0          1m
```

So, now multus is applied! Now what? 

## Time to run a pod!

Let's use our classic nginx pod, again. We're going to run this guy, and then we'll inspect some goodies.

Go ahead and make an `nginx_pod.yaml` file like so, then run a `kubectl create -f` against it.

```
[centos@kube-master ~]$ cat nginx_pod.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
[centos@kube-master ~]$ kubectl create -f nginx_pod.yaml 
```

Now watch the pods until you see the nginx containers running.

```
[centos@kube-master ~]$ watch -n1 kubectl get pods
```

What if you don't ever see the pods come up? Uh oh, that means something went wrong. As of today, this is all working swimmingly for me, but... It could go wrong. If that's the case, go ahead and describe one of the pods (remember, this `ReplicationController` yaml spins up 2 instances of nginx).

```
[centos@kube-master ~]$ kubectl describe pod nginx-vp516
```

## Let's inspect a pod and run `ip addr` to see the interfaces on it

Now that we have some pods up...  we can go ahead and check out what's inside them.

So pick either one of the nginx pods and let's execute the `ip addr` command in each one.

```
[centos@kube-master ~]$ kubectl exec nginx-vp516 -it ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:f4:01:02 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::11:99ff:fe68:c8ba/64 scope link tentative dadfailed 
       valid_lft forever preferred_lft forever
4: net0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    link/ether 0a:58:c0:a8:7a:c8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.200/24 scope global net0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:c0ff:fea8:7ac8/64 scope link 
       valid_lft forever preferred_lft forever
```

Woo hoo! That's great news. We've got 3 interfaces here. 

1. The loopback 
2. `eth0@if6` which is flannel.
3. `net0@if2` which is the bridge to eth0

You'll note that it's got an ip address assigned on the `192.168.122.0/24` network, in this case it's `192.168.122.200`. Awesomeness.

It's also got the IP address for the Flannel overlay. Which is `10.244.1.2` and matches what we see in a `kubectl describe pod`, like:

```
[centos@kube-master ~]$ kubectl describe pod nginx-vp516 | grep ^IP
IP:     10.244.1.2
```

Now that we've done that, we can go onto the virtual machine host, and we can curl that nginx instance!

So from the virtual machine host:

```
[user@virt-host ~]$ curl -s 192.168.122.200 | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

Awesomeness!

## Welcome to "easy mode"

Ok, so we just did all of that manually -- but, you can also use this playbook to do the "heavy lifting" (if it's that) for you. 

We'll assume you already kicked off the `virt-host-setup.yml` playbook, let's continue at the point where you've got your `./inventory/vms.inventory` all setup.

Basically the usual kube-install, but, we're going to specify that 

```
ansible-playbook -i inventory/vms.inventory kube-install.yml --extra-vars "pod_network_type=multus"
```

When that's complete, you should see 3 "kube-multus-ds" pods, one on each node in the cluster when you perform a `kubectl get pods --all-namespaces`. 

From there you can follow above steps to run a pod and verify that you've got the multiple network interfaces and what not.

Enjoy!!