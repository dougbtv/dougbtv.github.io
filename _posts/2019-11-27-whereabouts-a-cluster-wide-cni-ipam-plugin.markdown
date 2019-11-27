---
author: dougbtv
comments: true
date: 2019-11-27 08:00:00-05:00
layout: post
slug: whereabouts-a-cluster-wide-cni-ipam-plugin
title: Whereabouts -- A cluster-wide CNI IP Address Management (IPAM) plugin
category: nfvpe
---

Something that's a real challenge when you're trying to attach multiple networks to pods in Kubernetes is trying to get the right IP addresses assigned to those interfaces. Sure, you'd think, "Oh, give it an IP address, no big deal" -- but, turns out... It's less than trivial. That's why I came up with the IP Address Management (IPAM) plugin that I call "[Whereabouts](https://github.com/dougbtv/whereabouts)" -- you can think of it like a DHCP replacement, it assigns IP addresses dynamically to interfaces created by CNI plugins in Kubernetes. Today, we'll walk through how to use Whereabouts, and highlight some of the issues that it overcomes. First -- a little background.

The "multi-networking problem" in Kubernetes is something that's been near and dear to me. Basically what it boils down to is the question "How do you access multiple networks from networking-based workloads in Kube?" As a member of the [Network Plumbing Working Group](https://github.com/K8sNetworkPlumbingWG/community), I've helped to write a [specification](https://github.com/K8sNetworkPlumbingWG/multi-net-spec) for how to express your intent to attach to multiple networks, and I've contributed to [Multus CNI](https://github.com/intel/multus-cni) in the process. Multus CNI is a reference implementation of that spec and it gives you the ability to create additional interfaces in pods, each one of those interfaces created by CNI plugins. This kind of functionality is critical for creating network topologies that provide control and data plane isolation (for example). If you're a follower of my blog -- you'll know that I'm apt to use telephony examples (especially with Asterisk!) usually to show how you might isolate signal, media and control.

I'll admit to being somewhat biased (being a Multus maintainer), but typically I see community members pick up Multus and have some nice success with it rather quickly. However, sometimes they get tripped up when it comes to getting IP addresses assigned on their additional interfaces. Usually they start by [using the quick-start guide](https://github.com/intel/multus-cni/blob/master/doc/quickstart.md)). The examples for Multus CNI are focused on a quick start in a lab, and for IP address assignment, we use the [host-local reference plugin from the CNI maintainers](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/host-local). It works flawlessly for a single node.

![host-local with a single node](https://i.imgur.com/Wn1MTKo.png)

But... Once they get through the quickstart guide in a lab, they're like "Great! Ok, now let's exapand the scale a little bit..." and once that happens, they're using more than one node, and... It all comes crumbling down.

![host-local with multiple nodes](https://i.imgur.com/Uhmerd3.png)

See -- the reason why host local doesn't work across multiple nodes is actually right in the name "host-local" -- the storage for the IP allocations is local to each node. That is, it stores which IPs have been allocated in a flat file on the node, and it doesn't know if IPs in the same range have been allocated on a different node. This is... Frustrating, and really the core reasoning behind why I originally created Whereabouts. That's not to say there's anything inherently wrong with `host-local`, it works great for the purpose for which its designed, and its purview (from my view) is for local configurations for each node (which isn't necessarily the paradigm that's used with a technology like Multus CNI where CNI configurations aren't local to each node).

Of course, the next thing you might ask is "Why not just DHCP?" and actually that's what people typically try next. They'll try to use the [DHCP CNI plugin](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/dhcp). And you know, the DHCP CNI plugin is actually pretty great (and aside from the README, [these rkt docs](https://coreos.com/rkt/docs/latest/networking/overview.html) kind of explain it pretty well in the IP Address management section). But, some of it is less than intuitive. Firstly, it requires two parts -- one of which is to run the DHCP CNI plugin in "daemon mode". You've gotta have this running on each node, so you'll need a recipe to do just that. But... It's "DHCP CNI Plugin in Daemon Mode" it's not a "DHCP Server". Soooo -- if you don't already have a DHCP server you can use, you'll also need to setup a DHCP server itself. The "DHCP CNI Plugin in Daemon Mode" just gives you a way to listen to for DHCP messages.

And personally -- I think managing a DHCP server is a pain in the gluteous maximus. And it's the beginning of ski season, and I'm a [telemark](https://en.wikipedia.org/wiki/Telemark_skiing) skier, so I have enough of those pains.

I'd also like to give some BIG THANKS! I'd like to point out that [Christopher Randles](https://github.com/crandles) has made some monstrous contributions to Whereabouts -- especially but not limited to the engine which provides the Kubernetes-backed data store (Thanks Christopher!). Additionally, I'd also like to thank [Tomofumi Hayashi](https://github.com/s1061123) who is the author of the [static IPAM CNI plugin](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/static). I originally based Whereabouts on the structure of the static IPAM CNI plugin as it had all the basics, and also I could leverage what was built there to allow Whereabouts users to also use the static features alongside Whereabouts.

## How Whereabouts works

![How Whereabouts Works](https://i.imgur.com/RSefMjZ.png)

From a user perspective, it's pretty easy -- basically, you add a section to your CNI configuration(s). [The CNI specification has a construct for "ipam" -- IP Address management](https://github.com/containernetworking/cni/blob/master/SPEC.md#ip-allocation). 

Here's an example of what a Whereabouts configuration looks like:

```
"ipam": {
    "type": "whereabouts",
    "datastore": "kubernetes",
    "kubernetes": { "kubeconfig": "/etc/cni/net.d/whereabouts.d/whereabouts.kubeconfig" },
    "range": "192.168.2.0/24"
  }
```

Here, we're essentially saying:

* We choose `whereabouts` as a value for `type` which defines which IPAM plugin we're calling.
* We'd like to use `kubernetes` for our `datastore` (where we'll store the IP addresses we've allocated) (and we'll provide a `kubeconfig` for it, so Whereabouts can access the kube API)
* And we'd like an IP address `range` that's a `/24` -- we're asking Whereabouts to assign us IP addresses in the range of `192.168.2.1` to `192.168.2.255`.

Behind the scenes, honestly... It's not much more complex than what you might assume from the exposed knobs from the user perspective. Essentially -- it's storing the IP address allocations in a data store. It can use the Kubernetes API natively to do so, or, it can use an etcd instance. This provides a method to access what's been allocated across the cluster -- so you can assign IP addresses across nodes in the cluster (unlike being limited to a single host, with `host-local`). Otherwise, regarding internals -- I have to admit it was kind of satisfying to program the logic to scan through IP address ranges with bitwise operations, ok I'm downplaying it... Let's be honest, it was super satisifying.

## Requirements

* A Kubernetes Cluster v1.16 or later
    - You can use a lesser version of Kubernetes, but, you might have to tweak some deployments.
    - I'd recommend 2 or more worker nodes, to make it more interesting.
    - If you need to install it, may I recommend my ["Choose your own adventure" article on Kubernetes installation](https://dougbtv.com/nfvpe/2018/03/21/kubernetes-on-centos/).
* You need a default network CNI plugin installed (like Flannel [or Weave, or Calico, etc, etc])
* Multus CNI
    - I'll cover a basic installation here, so you don't need to have it right now. But, if you already have it installed, you'll save a step.
    - If you're using OpenShift -- you already have all of the above out of the box, so you're all set.

Essentially, all of the commands will be run from wherever you have access to `kubectl`.

## Let's install Multus CNI

You can always refer to the [quick start guide](https://github.com/intel/multus-cni/blob/master/doc/quickstart.md) if you'd like more information about it, but, I'll provide the cheat sheet here.

Basically we just clone the Multus repo and then apply the daemonset for it...

```
git clone https://github.com/intel/multus-cni.git && cd multus-cni
cat ./images/multus-daemonset.yml | kubectl apply -f -
```

You can check to see that it's been installed by watching the pods for it come up, with `watch -n1 kubectl get pods --all-namespaces`. When you see the `kube-multus-ds-*` pods in a `Running` state you're good. If you're a curious type you can check out the contents (on any or all nodes) of `/etc/cni/net.d/00-multus.conf` to see how Multus was configured.

## Let's fire up Whereabouts!

The installation for it is easy, it's basically the same as Multus, we clone it and apply the daemonset. This is copied directly from the [Whereabouts README](https://github.com/dougbtv/whereabouts).

```
git clone https://github.com/dougbtv/whereabouts && cd whereabouts
kubectl apply -f ./doc/daemonset-install.yaml -f ./doc/whereabouts.cni.k8s.io_ippools.yaml
```

Same drill as above, just wait for the pods to come up with `watch -n1 kubectl get pods --all-namespaces`, they're named `whereabouts-*` (usually in the kube-system namespace).

## Time for a test drive

The goal here is to create a configuration to add an extra interface on a pod, add a Whereabouts configurations to that, spin up two pods, have those pods on different nodes, and show that they've been assigned IP addresses as we've specified.

Alright, what I'm going to do next is to give my nodes some labels so I can be assured that pods wind up on different nodes -- this is mostly just used to illustrate that Whereabouts works with multiple nodes (as opposed to how `host-local` works).

```
$ kubectl get nodes
$ kubectl label node kube-whereabouts-demo-node-1 side=left
$ kubectl label node kube-whereabouts-demo-node-2 side=right
$ kubectl get nodes --show-labels
```

Now what we're going to do is create a `NetworkAttachmentDefinition` -- this a custom resource that we'll create to express that we'd like to attach an additional interface to a pod. Basically what we do is pack a CNI configuration inside our `NetworkAttachmentDefinition`. In this CNI configuration we'll also include our whereabouts config.

Here's how I created mine:

```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "name": "whereaboutsexample",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "datastore": "kubernetes",
        "kubernetes": { "kubeconfig": "/etc/cni/net.d/whereabouts.d/whereabouts.kubeconfig" },
        "range": "192.168.2.225/28",
        "log_file" : "/tmp/whereabouts.log",
        "log_level" : "debug"
      }
    }'
EOF
```

What we're doing here is creating a `NetworkAttachmentDefinition` for a [macvlan](https://sreeninet.wordpress.com/2016/05/29/macvlan-and-ipvlan/)-type interface (using the [macvlan CNI plugin](https://github.com/containernetworking/plugins/tree/master/plugins/main/macvlan)).

NOTE: If you're copying and pasting the above configuration (and I hope you are!) make sure you set the `master` parameter to match the name of a real interface name as available on your nodes.

Then we specify an `ipam` section, and we say that we want to use `whereabouts` as our `type` of IPAM plugin. We specify where the kubeconfig lives (this gives whereabouts access to the Kube API). 

And maybe most important to us as users -- we specify the range we'd like to have IP addresses assigned in. You can use CIDR notation here, and... If you need to use other options to exclude ranges, or other range formats -- check out [the README's guide to the core parameters](https://github.com/dougbtv/whereabouts#core-parameters).

After we've created this configuration, we can list it too -- in case we need to remove or change it later, such as:

```
$ kubectl get network-attachment-definitions.k8s.cni.cncf.io
```

Alright, we have all our basic setup together, now let's finally spin up some pods...

Note that we have `annotations` here that include `k8s.v1.cni.cncf.io/networks: macvlan-conf` -- that value of `macvlan-conf` matches the name of the `NetworkAttachmentDefinition` that we created above.

Let's create the first pod for our "left side" label:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod-left
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: samplepod-left
    command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: dougbtv/centos-network
  nodeSelector:
    side: left
EOF
```

And again for the right side:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod-right
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: samplepod-right
    command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: dougbtv/centos-network
  nodeSelector:
    side: right
EOF
```

I then wait for the pods to come up with `watch -n1 kubectl get pods --all-namespaces` or I look at the details of one pod with `watch -n1 'kubectl describe pod samplepod-left | tail -n 50'`

Also -- you'll note if you `kubectl get pods -o wide` the pods are indeed running on different nodes.

Once the pods are up and in a `Running` state, we can interact with them.

The first thing I do is check out that the IPs have been assigned:

```
$ kubectl exec -it samplepod-left -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 3e:f7:4b:a1:16:4b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.2.4/24 scope global eth0
       valid_lft forever preferred_lft forever
4: net1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether b6:42:18:70:12:6e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.2.225/28 scope global net1
       valid_lft forever preferred_lft forever
```

You'll note there's three interfaces, a local loopback, an `eth0` that's for our "default network" (where we have pod-to-pod connectivity by default), and an additional interface -- `net1`. This is our macvlan connection AND it's got an IP address assigned dynamically by Whereabouts. In this case `192.168.2.225`

Let's check out the right side, too:

```
$ kubectl exec -it samplepod-right -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 96:28:58:b9:a4:4c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.1.3/24 scope global eth0
       valid_lft forever preferred_lft forever
4: net1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether 7a:31:a7:57:82:1f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.2.226/28 scope global net1
       valid_lft forever preferred_lft forever
```

Great, we've got another dynamically assigned address that does not collide with our already reserved IP address from the left side! Our address on the right side here is `192.168.2.226`.

And while connectivity is kind of outside the scope of this article -- in most cases it should generally work right out the box, and you should be able to ping from one pod to the next!

```
[centos@kube-whereabouts-demo-master whereabouts]$ kubectl exec -it samplepod-right -- ping -c5 192.168.2.225
PING 192.168.2.225 (192.168.2.225) 56(84) bytes of data.
64 bytes from 192.168.2.225: icmp_seq=1 ttl=64 time=0.438 ms
64 bytes from 192.168.2.225: icmp_seq=2 ttl=64 time=0.217 ms
64 bytes from 192.168.2.225: icmp_seq=3 ttl=64 time=0.316 ms
64 bytes from 192.168.2.225: icmp_seq=4 ttl=64 time=0.269 ms
64 bytes from 192.168.2.225: icmp_seq=5 ttl=64 time=0.226 ms
```

And that's how you can determine your pods Whereabouts (by assigning it a dynamic address without the pain of runnning DHCP!).
