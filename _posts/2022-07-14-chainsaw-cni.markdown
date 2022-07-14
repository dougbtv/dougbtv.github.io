---
author: dougbtv
comments: true
date: 2022-07-14 08:00:00-05:00
layout: post
slug: chainsaw-cni
title: Chainsaw CNI -- Modify container networking at runtime
category: nfvpe
---

# Introducing: [Chainsaw CNI](https://github.com/dougbtv/chainsaw-cni)

The gist of Chainsaw CNI (brum-brum-brum-brum-brrrrrrrrr) is it's a CNI plugin that runs in a CNI chain (more on that soon), and it allows you to run arbitrary [`ip` commands](https://man7.org/linux/man-pages/man8/ip.8.html) against your Kubernetes pods to either manipulate or inspect networking. You can do this at run-time by annotating a pod with the commands you want to run.

For example, you can annotate a pod with:

```
k8s.v1.cni.cncf.io/chainsaw: >
      ["ip route","ip addr"]
```

And then get the output of `ip route` and `ip addr` for your pod.

I named it Chainsaw because:

* It works using CNI Chains.
* It's powerful, but kind of dangerous.

Today, we're going to:

* Talk about why I made it.
* Look at what CNI chains are.
* See what the architecture is comprised of.
* And of course, engage the choke, pull the rope start and fire up this chainsaw.

We'll be using it with network attachment definitions -- that is, the custom resource type that's used by [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni)

Why do you say it's dangerous? Well, like a chainsaw, you do permanent harm to something. You could totally turn off networking for a pod. Or, potentially you open up a way for some user of your system to do something more privileged than you thought. I'm still thinking about how to better address this part, but for now... I'd advise that you use it carefully, and in lab situations rather than production before these aspects are more fully considered.

Also, as an aside... I am a physical chainsaw user. I have one and, I use it. But I'm appropriately afraid of it. I take a long long time to think about it before I use it. I've watched a bunch of videos about it, but I really want to take a [Game Of Logging course](http://www.gameoflogging.com/) so I can really operate it safely. Typically, I'm just using my [Silky Katanaboy](https://silkysaws.com/silky-katanaboy-500-folding-saw/) (awesome Japanese pull saw!) for trail work and what not.

Last but not least, a quick disclaimer: This is... a really new project. So it's missing all kinds of stuff you might take for granted: unit tests, automatic builds, all that. Just a proof of concept, really.

## Why, though?

I was originally inspired by this hearing this particular discussion:

Person: *"Hey I want to manipulate a route on a particular pod"*

Me: *"Cool, that's totally possible, use the [route override CNI](https://github.com/openshift/route-override-cni)" (it's another chained plugin!)*

Person: *"But I don't want to manipulate the net-attach-def, there's **tons** of pods using them, and I only want to manipulate for a specific site, so I want to do it at runtime, adding more net-attach-defs makes life harder".*

Well, this kinda bothered me! I talked to a co-worker who said *"Sure, next they're going to want to change EVERYTHING at runtime!"*

I was thinking: *"hey, what if you COULD change whatever you wanted at runtime?"*

And I figured, it could be a really handy tool, even if just for CNI developers, or network tinkerers as it may be.

## CNI Chains

```

   ┌──────────────────┐                   ┌────────────────┐
   │                  │                   │                │
   │                  │   ┌───────────┐   │                │
   │   CNI Plugin A   │   │           │   │  CNI Plugin B  │
   │                  ├───► cni result├───►                │
   │                  │   │           │   │                │
   │                  │   └───────────┘   │                │
   └──────────────────┘                   └────────────────┘

```

CNI chains are... sometimes confusing to people. But, they don't need to be, it's basically as simple as saying, "You can chain as many CNI plugins together as you want, and each CNI plugin gets all the CNI results of the plugin before it"

This functionality was [introduced in CNI 0.3.0](https://github.com/containernetworking/cni/blob/main/SPEC.md#released-versions) and is available in all later versions of CNI, naturally.

You can tell if you have a CNI plugin chain by looking at your CNI configuration, if the top level JSON has the `"type"` field -- then it's not a chain.

If it has the `"plugins": []` array -- then it's a chain of plugins, and will run in the order within the array. As of CNI 1.0, you'll *always* be using the plugins field, and always have chains, even if a "chain of one".

Why do you use chained plugins? The best example I can usually think of is the [Tuning Plugin](https://www.cni.dev/plugins/current/meta/tuning/). Which allows you to set network [sysctls](https://en.wikipedia.org/wiki/Sysctl), or manipulate other parameters of networks -- such as setting an interface into [promiscuous mode](https://en.wikipedia.org/wiki/Promiscuous_mode). This is done typically after the work of your main plugin, which is going to do the plumbing to setup the networking for you (e.g. say, a vxlan tunnel, or a macvlan interface, etc etc).

## The architecture

Not a whole lot to say, but it's a "sort of thick plugin" -- thick CNI plugins are those that have a resident daemon, as opposed to "thin CNI plugins" -- which run as a one-shot (all of [the reference CNI plugins](https://github.com/containernetworking/plugins) are one shots). But in this case, we just use the daemonset that's resident for looking at the log output, for inspecting our results.

Other than that, it's similar to Multus CNI in that it knows how to talk to the k8s API and get the annotations, and it uses a generated kubeconfig to authorize itself against the k8s API

## Let's get to using it!

Requirements:

* A k8s cluster, the newer the beter.
* [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) must be installed

That's about it. Don't use a production cluster ;)

So go ahead and clone [dougbtv/chainsaw-cni](https://github.com/dougbtv/chainsaw-cni).

Then create the daemonset with:

```
kubectl create -f deployments/daemonset.yaml
```

*NOTE*: Are you an openshift user? Use the `deployments/daemonset_openshift.yaml` deployment instead `:thumbsup:`

Now, let's create a net-attach-def which implements chainsaw in a chain -- note the `plugins` array!

Also note the use of the special token `CURRENT_INTERFACE` which will use the current interface name as opposed to you having to know it in advance.

```
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: test-chainsaw
spec:
  config: '{
    "cniVersion": "0.4.0",
    "name": "test-chainsaw-chain",
    "plugins": [{
      "type": "bridge",
      "name": "mybridge",
      "bridge": "chainsawbr0",
      "ipam": {
        "type": "host-local",
        "subnet": "192.0.2.0/24"
      }
    }, {
      "type": "chainsaw",
      "foo": "bar"
    }]
  }'
---
apiVersion: v1
kind: Pod
metadata:
  name: chainsawtestpod
  annotations:
    k8s.v1.cni.cncf.io/networks: test-chainsaw
    k8s.v1.cni.cncf.io/chainsaw: >
      ["ip route add 192.0.3.0/24 dev CURRENT_INTERFACE", "ip route"]
spec:
  containers:
  - name: chainsawtestpod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine
```

Next, check what node the pod is running with:

```
kubectl get pods -o wide
```

You can then find the output from the results of the ip commands from the chainsaw daemonset that is running on that node, e.g.

```
kubectl get pods -n kube-system -o wide | grep -iP "status|chainsaw"
```

And looking at the logs for the daemonset pod that correlates to the node on which the pod resides, for example:

```
kubectl logs kube-chainsaw-cni-ds-kgx69 -n kube-system
```

You'll see that we have added a route to `192.0.3.0/24` and then show the IP route output!

So my results look like:

```
Detected commands: [route add 192.0.3.0/24 dev CURRENT_INTERFACE route]
Running ip netns exec 901afa16-48e7-4f22-b2b1-7678fa3e9f5e ip route add 192.0.3.0/24 dev net1 ===============


Running ip netns exec 901afa16-48e7-4f22-b2b1-7678fa3e9f5e ip route ===============
default via 10.129.2.1 dev eth0 
10.128.0.0/14 dev eth0 
10.129.2.0/23 dev eth0 proto kernel scope link src 10.129.2.64 
172.30.0.0/16 via 10.129.2.1 dev eth0 
192.0.2.0/24 dev net1 proto kernel scope link src 192.0.2.51 
192.0.3.0/24 dev net1 scope link 
224.0.0.0/4 dev eth0 
```
