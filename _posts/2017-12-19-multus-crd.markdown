---
author: dougbtv
comments: true
date: 2017-12-19 11:00:01-05:00
layout: post
slug: multus-crd
title: Kubernetes multiple network interfaces -- but! With different configs per pod; Multus CNI has your back.
category: nfvpe
---

You need multiple network interfaces in each pod -- because you, like me, have some more serious networking requirements for Kubernetes than your average bear. The thing is -- if you have different specifications for each pod, and what network interfaces each pod should have based on its role, well... Previously you were fairly limited. At least using [my previous (and somewhat dated) method](http://dougbtv.com/nfvpe/2017/02/22/multus-cni/) of using [Multus CNI](https://github.com/Intel-Corp/multus-cni) (a CNI plugin to help enable you to have multiple interfaces per pod), you could only apply to all pods (or at best, with multiple CNI configs per box, and have it per box). Thanks to [Kural](https://github.com/rkamudhan) and crew, Multus includes the functionality to use [Kubernetes Custom Resources](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) (Also known as "CRDs"). These "custom resource definitions" are a way to extend the Kubernetes API. Today we'll take advantage of that functionality. The CRD implementation in Multus allows us to specify exactly what multiple network interfaces each pod has based on annotations attached to each pod. Our goal here will be to spin up a Kubernetes cluster complete with Multus CNI (including the CRD functionality), and then we'll spin up pods where we have some with a single interface, and some with multiple interfaces, and then we'll inspect those. 

Not familiar with Multus CNI? The short version is that it's (in my own words) a "meta plugin" -- one that lets you call multiple CNI plugins, and assign an interface in a pod to each of those plugins. This allows us to create multiple interfaces.

Have an older Kubernetes? At the time of writing Kubernetes 1.9.0 was hot off the presses. So CRDs are well established, but if you have an older edition Multus also supports "TPRs" -- third party resources, which were an earlier incantation of what is now CRDs. You'll have to modify for those to work, but, this might be a fair reference point.

A lot of what I learned here is directly from the [Multus CNI readme](https://github.com/Intel-Corp/multus-cni). Mostly I have just automated it with kube-ansible, and then documented up my way of doing it. Make sure to check out what's in the official readme to further extend your knowledge of what you can do with Multus.

In short, what's cool about this?

* Multus CNI can give us multiple interfaces per each Kubernetes pod
* The CRD functionality for Multus can allow us to specify which pods get which interfaces, and allowing different interfaces depending on the use case.

I originally really wanted to do something neat with a realistic use-case. Like separate networks like I used to do frequently for telephony use cases. In those cases I'd have different network segments for management, signalling and media. I was going to setup a neat VoIP configuration here, but, alas... I kept [yak shaving](https://en.wiktionary.org/wiki/yak_shaving) to get there. So instead, we'll just get to the point and today we're just going to spin up some example pods, and maybe next time around I'll have a more realistic use-case rather than just saying "There it is, it works!". But, today, it's just "there it is!"

## Requirements

TL;DR:

* A CentOS 7 box capable of running some virtual machines.
* Ansible installed on a workstation.
* Git.
* Your favorite text editor.
* Some [really good coffee](http://www.vermontcoffeecompany.com/buy_vermont_organic_fair_trade_coffee.html).
  - Tea is also a fair substitute, but, if herbal -- it must be a rooibos.

This tutorial will use [kube-ansible](https://github.com/redhat-nfvpe/kube-ansible), which is an Ansible playbook that I reference often in this blog, but, it's a way to spin up a Kubernetes cluster (on CentOS) with vanilla kubernetes in order to create a Kubernetes development cluster for yourself quickly, and including some scenarios.

In this case we're going to spin up a couple virtual machines and deploy to those. You don't need a high powered machine for this, just enough to get a couple light VMs to use for our experiment.

## Get your clone on.

Go ahead and clone kube-ansible, and move into its directory.

```
$ git clone -b v0.1.8 git@github.com:redhat-nfvpe/kube-ansible.git && cd kube-ansible/
```

Install the required galaxy roles for the project.

```
$ ansible-galaxy install -r requirements.yml
```

## Setup your inventory and extra vars.

Make sure you can SSH to the CentOS 7 machine we'll use as a virtualization host (referred to heavily as "virthost" in the Ansible playbooks, and docs, and probably here in this article). Then create yourself an inventory for that host. For a reference, here's what mine looks like:

```
$ cat inventory/virthost.inventory 
the_virthost ansible_host=192.168.1.119 ansible_ssh_user=root

[virthost]
the_virthost
```

We're also going to create some extra variables to use. So let's define those.

Pay attention to these parts:

* `bridge_` variables define how we'll bridge to the network of your virthost. In this case I want to bridge to the device called `enp1s0f1` on that host, which I specify as `bridge_physical_nic`. I then specify a `bridge_network_cidr` which matches the DHCP range on that network (in this example case I have a SOHO type setup with a `192.168.1.0/24` subnet.)
* `multus_ipam_` variables define how we're going to use some networking with a plugin (it'll be `macvlan`, a little more on that later) that this playbook automatically sets up for us. Generally this should match what your network looks like, so in my SOHO type example, we have a gateway on `192.168.1.1` and then we match that.

The rest of the variables can likely stay the same.

```
$ cat inventory/multus-extravars.yml 
---
bridge_networking: true
bridge_name: virbr0
bridge_physical_nic: "enp1s0f1"
bridge_network_name: "br0"
bridge_network_cidr: 192.168.1.0/24
pod_network_type: "multus"
virtual_machines:
  - name: kube-master
    node_type: master
  - name: kube-node-1
    node_type: nodes
optional_packages:
  - tcpdump
  - bind-utils
multus_use_crd: true
multus_ipam_subnet: "192.168.1.0/24"
multus_ipam_rangeStart: "192.168.1.200"
multus_ipam_rangeEnd: "192.168.1.216"
multus_ipam_gateway: "192.168.1.1"
```

## Initial setup the virtualization host

Cool, with those in place, we can now begin our initial virthost setup. Let's run that with the inventory and extra vars we just created.

```
$ ansible-playbook -i inventory/virthost.inventory -e "@./inventory/multus-extravars.yml" virthost-setup.yml
```

This has done a few things for us: It has spun up some virtual machines, and  created a local inventory of those virtual machines, and also it has put a ssh key in `~/.ssh/the_virthost/id_vm_rsa` -- which we can use if we want to SSH to one of those hosts. (Which we'll do here in a minute)

Now, let's kick off a deployment of Kubernetes, it will also get. This is the part of the tute where you'll need that coffee I mentioned earlier.

```
$ ansible-playbook -i inventory/vms.local.generated -e "@./inventory/multus-extravars.yml" kube-install.yml 
```

Finished your coffee yet? Ok, heat it up, we're going to enter a machine and take a look around.

## Overview of what's happened.

I highly suggest you take a peek around the Ansible playbooks if you want some details of what has happened for you. Sure, they're pretty big, but, you don't need to be an Ansible genius to figure out what's going on. 

As a quick recap, here's some of the things the playbook has done for us:

* Installed the basic packages we need for Kubernetes
* Initialized a Kubernetes cluster using [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
* Compiled Multus CNI
* Configured some RBAC so that our nodes can query the Kubernetes API (which Multus needs in order to use CRDs)
* Added some CRDs to our setup that Multus can use to figure out which pods get which treatments for their network configuration.

## Inspecting the setup.

Here's one way that you can use to ssh to the master...

```
$ ssh -i ~/.ssh/the_virthost/id_vm_rsa centos@$(grep -m1 "kube-master" inventory/vms.local.generated | cut -d"=" -f 2)
```

You might first want to checkout the health of the cluster with a `kubectl get nodes` and make sure that it's generally functioning OK. In this case we're building a cluster with a single master, and a single node.

Let's peek around at a few things that the playbook has setup for us... Before anything else -- the CNI config.

```
[centos@kube-master centos]$ sudo cat /etc/cni/net.d/10-multus.conf 
{
  "name": "multus-cni-network",
  "type": "multus",
  "kubeconfig": "/etc/kubernetes/kubelet.conf"
}
```

You'll see that it just has a skeleton for Multus. The real configs will really be in CRD.

## The Custom Resource Definitions (CRDs)

Check this out -- we have a CRD, `networks.kubernetes.com`:

```
[centos@kube-master ~]$ kubectl get crd
NAME                      AGE
networks.kubernetes.com   46m
```

We can also `kubectl` that, too.

```
[centos@kube-master ~]$ kubectl get networks
NAME           AGE
flannel-conf   46m
macvlan-conf   46m
```

Great, now let's describe one of the networks...

```
[centos@kube-master ~]$ kubectl describe networks flannel-conf
Name:         flannel-conf
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  kubernetes.com/v1
Args:         [ { "delegate": { "isDefaultGateway": true } } ]
[...snip ...]
```

You can also describe the `macvlan-conf`, too. With `kubectl describe networks macvlan-conf`.

So check this out, there's a really really simple CNI configuration there in the `Args:`. It's just a simple config that points to flannel. That's it.

## Spin up a pod!

That being the case, let's setup a pod from this spec.

```
[centos@kube-master ~]$ cat flannel.pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: flannelpod
  annotations:
    networks: '[  
        { "name": "flannel-conf" }
    ]'
spec:
  containers:
  - name: flannelpod
    command: ["/bin/bash", "-c", "sleep 2000000000000"]
    image: dougbtv/centos-network
    ports:
    - containerPort: 80
```

Create that pod spec YAML however you'd like, and then we'll create from it.

```
[centos@kube-master ~]$ kubectl create -f flannel.pod.yaml 
pod "flannelpod" created
```

Watch it come up if you wish, with `watch -n1 kubectl get pods -o wide`. Or even get some detail with `watch -n1 kubectl describe pod flannelpod`

Now, let's look at the interfaces therein... In this case, we have a vanilla flannel setup for this pod. There's a `lo` loopback device interface, and then `eth0` which has an IP address assigned on the `10.244.1.2` address in the CIDR range the playbooks setup for us.

```
[centos@kube-master ~]$ kubectl exec -it flannelpod -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 0a:58:0a:f4:01:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.1.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a8a0:b3ff:febd:4e0a/64 scope link 
       valid_lft forever preferred_lft forever
```

## How about... another pod!

Well naturally, this wouldn't be a very good demonstration if we didn't show you how you could create yet another pod -- but with a different set of networks using CRD. So, let's get on with it and create another!

This time, you'll note that the annotation is different here, instead of `flannel-conf` in the `networks` in `annotations` we have `macvlan-conf` which you'll notice correlates with the object we have created (via the playbooks) in the CRDs.

Here's my example pod spec...

```
[centos@kube-master ~]$ cat macvlan.pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: macvlanpod
  annotations:
    networks: '[  
        { "name": "macvlan-conf" }
    ]'
spec:
  containers:
  - name: macvlanpod
    command: ["/bin/bash", "-c", "sleep 2000000000000"]
    image: dougbtv/centos-network
    ports:
    - containerPort: 80
```

And I create that...

```
kubectl create -f macvlan.pod.yaml 
```

And then I watch that come up too (much quicker this time as it in theory should've pulled the image already to the same node)

```
$ watch -n1 kubectl describe pod macvlanpod
```

Now let's check out the `ip a` on that pod, too.

```
[centos@kube-master ~]$ kubectl exec -it macvlanpod -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether 96:ea:41:2b:38:23 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.200/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::94ea:41ff:fe2b:3823/64 scope link 
       valid_lft forever preferred_lft forever
```

Cool! It's got an address on the `192.168.1.0/24` network. In theory, you could ping this pod from elsewhere on that network. In my case, I'm going to open up a ping stream to this pod on my workstation (which is VPN'd in and presents as `192.168.1.199`) and then I'm going to sniff some packets with `tcpdump` while I'm at it.

On my workstation...

```
$ ping -c 100 192.168.1.200
```

And then from the pod...

```
[centos@kube-master ~]$ kubectl exec -it macvlanpod -- /bin/bash
[root@macvlanpod /]# tcpdump -i any icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 65535 bytes
18:30:21.195765 IP 192.168.1.199 > macvlanpod: ICMP echo request, id 695, seq 43, length 64
18:30:21.195814 IP macvlanpod > 192.168.1.199: ICMP echo reply, id 695, seq 43, length 64
18:30:22.197676 IP 192.168.1.199 > macvlanpod: ICMP echo request, id 695, seq 44, length 64
18:30:22.197721 IP macvlanpod > 192.168.1.199: ICMP echo reply, id 695, seq 44, length 64
```

Hey did you notice anything yet? There's not truly multi-interface!

## Hey you duped me, this isn't multi-interface!

Ah ha! Now this is the part where we'll bring it all together my good friend. Let's create a pod that has BOTH macvlan, and flannel... All we have to do is create a list in the annotations -- the astute eye may have noticed that the JSON already had the brackets for a list.

```
$ cat both.pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: bothpod
  annotations:
    networks: '[  
        { "name": "macvlan-conf" },
        { "name": "flannel-conf" }
    ]'
spec:
  containers:
  - name: bothpod
    command: ["/bin/bash", "-c", "sleep 2000000000000"]
    image: dougbtv/centos-network
    ports:
    - containerPort: 80
```

And create with that...

```
kubectl create -f both.pod.yaml
```

Of course, I watch it come up with `watch -n1 kubectl describe pod bothpod`.

And I can see that there's now multiple interfaces -- loopback, flannel, and macvlan!

```
[centos@kube-master multus-resources]$ kubectl exec -it bothpod -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether c6:bc:74:df:80:7b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.201/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c4bc:74ff:fedf:807b/64 scope link 
       valid_lft forever preferred_lft forever
4: net0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 0a:58:0a:f4:01:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.1.3/24 scope global net0
       valid_lft forever preferred_lft forever
    inet6 fe80::6c4e:c5ff:fe5d:64f8/64 scope link 
       valid_lft forever preferred_lft forever
```

Here you can see it shows both the `10.` network for flannel (net0), and the `192.168.1.0/24` network for the macvlan plugin (eth0).

Thanks for giving it a try! If you run into any issues, make sure to post 'em on the issues for the [kube-ansible github](https://github.com/redhat-nfvpe/kube-ansible), or, if they're multus specific (and not setup specific) to [Multus CNI](https://github.com/Intel-Corp/multus-cni) repo.
