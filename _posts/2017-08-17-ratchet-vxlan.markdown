---
author: dougbtv
comments: true
date: 2017-08-17 14:30:00-05:00
layout: post
slug: ratchet-vxlan
title: Ratchet CNI: Using VXLAN for container isolation in Kubernetes
category: nfvpe
---

In today's episode we're looking at [Ratchet CNI](https://github.com/dougbtv/ratchet-cni), an implementation of [Koko](https://github.com/redhat-nfvpe/koko) -- but in CNI, the container networking interface that is used by Kubernetes for creating network interfaces. The idea being that the network interface creation can be performed by Kubernetes via CNI. Specifically we're going to create some network isolation of network links between containers to demonstrate a series of "cloud routers". We can use the capabilities of Koko to both create vEth connections between containers when they're local to the same host, and then VXLAN tunnels to containers when they're across hosts. Our goal today will be to install & configure Ratchet CNI on an existing cluster, we'll verify it's working, and then we'll install a cloud router setup based on [zebra pen](https://github.com/dougbtv/zebra-pen) (a cloud router demo).

Here's what the router will look like when we're done:

![diagram](http://i.imgur.com/rpO2A20.png)

The gist is that the green boxes are Kubernetes minions which run pods, and the blue boxes are pods running on those hosts, and the yellow boxes are the network interfaces that will be created by Ratchet (and therefore Koko). In this scenario, just one VXLAN tunnel is created when going between the hosts.

Note that Ratchet is still a prototype, and some of the constraints of it are limited to the static way in which interfaces and addressing is specified. This is indeed a limitation, but is intended to illustrate how you might specify the links between these containers.

## Requirements

Required:

* A Kube cluster, [spin one up my way](https://github.com/dougbtv/kube-centos-ansible) if you wish.
* Two nodes where you can schedule pods (and have the ability modify the CNI configuration on those nodes)

Optional:

* An operational Flannel plugin in CNI running on the cluster beforehand.

It's worth it for you to note that I use an all CentOS 7 based cluster, which while it isn't required, definitely colors how I use the ancillary tools and approach things.

## Installing the Ratchet binaries

First thing we're going to do is on each of the nodes we're going to use here. In my case it's just going to be two minion nodes which can schedule pods, I don't bother putting it on my master.

Here's how I download and put the binaries into place:

```
$ curl -L https://github.com/dougbtv/ratchet-cni/releases/download/v0.1.0/ratchet-cni-v0.1.0.tar.gz > ratchet.tar.gz
$ tar -xzvf ratchet.tar.gz 
$ sudo mv ratchet-cni-v0.1.0/* /opt/cni/bin/
```

That's it -- you've got ratchet! (Again, man, Go makes it easy, right.)

## Spin up etcd

You'll need to have an etcd instance -- if you have a running instance you want to use for this, go ahead. I'll include my scheme here where I run my own.

From wherever you have `kubectl` available, go ahead and work on these steps.

Firstly, I create a new namespace to run these etcd pods in...

```
$ tee ratchet-namespace.yaml <<'EOF'
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "ratchet",
    "labels": {
      "name": "ratchet"
    }
  }
}
EOF
$ kubectl create -f ratchet-namespace.yaml 
$ kubectl get namespaces
```

I have an [example etcd pod spec in this gist](https://gist.github.com/dougbtv/29bcd019e3229be70bc58e1ce85da5ea), I download that...

```
[centos@kube-master ~]$ curl -L https://goo.gl/eMnsh9 > etcd.yaml
```

And then create it in the ratchet namespace we just created, and watch it come up.

```
$ kubectl create -f etcd.yaml --namespace=ratchet
$ watch -n1 kubectl get pods --namespace=ratchet
```

This has also created a service for us.

```
[centos@kube-master ~]$ kubectl get svc --namespace=ratchet | head -n2
NAME          CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
etcd-client   10.102.72.174   <none>        2379/TCP            56s
```

This service is important to the Ratchet configuration. So note how you can access this service -- you can use the IP if all else fails, at least for testing that's just fine. You don't want to rely on that full-time, however.

If your nodes don't resolve `etcd-client.ratchet.svc.cluster.local` -- pay special attention. As this is the DNS name for etcd I'll use in the following configs.

## Configuring Ratchet

Now we need to put configurations into place. Firstly, you're going to want to clear out whatevers in `/etc/cni/net.d/`, I recommend before getting to this point that you have flannel working because we can do something cool with this plugin available -- we can bypass ratchet and pass along ineligible pods to Flannel (or any other plugin). I'll include configs that have Flannel, here. If appropriate, replace with another plugin configuration.

Here I am moving my configs to a backup directory, do this on both hosts that will run Ratchet...

```
[centos@kube-minion-1 ~]$ mkdir cni-configs
[centos@kube-minion-1 ~]$ sudo mv /etc/cni/net.d/* ./cni-configs/
```

Let's look at my current configuration...

```
[centos@kube-minion-2 ~]$ cat cni-configs/10-flannel.conf 
{
  "name": "cbr0",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
}
```

It's a Flannel config, I'm gonna keep this around for a minute, cause I'll use it in my upcoming configs.

Next, let's assess what you have available for networking. Mine is pretty simple. Each of my nodes have a single nic -- `eth0`, and it's on the `192.168.1.0/24` network, and that network is essentially flat -- it can access on that NIC, and also the other nodes on the network. Naturally, in real life -- your network will be more complex. But, in this step... Choose the proper NIC and IP address for your setup.

So, I pick out my NIC and IP address, what's it look like on my nodes...

```
[centos@kube-minion-1 ~]$ ip a | grep -Pi "eth0|inet 192"
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 192.168.1.73/24 brd 192.168.1.255 scope global dynamic eth0
```

Ok, cool, so I have `eth0` and it's `192.168.1.73` -- these are both going into my Ratchet config.

Now, here's my Ratchet config I've created on this node, as `/etc/cni/net.d/10-ratchet.conf`:

```
[centos@kube-minion-1 ~]$ cat /etc/cni/net.d/10-ratchet.conf
{
  "name": "ratchet-demo",
  "type": "ratchet",
  "etcd_host": "etcd-client.ratchet.svc.cluster.local",
  "etcd_port": "2379",
  "child_path": "/opt/cni/bin/ratchet-child",
  "parent_interface": "eth0",
  "parent_address": "192.168.1.73",
  "use_labels": true,
  "delegate": {
    "name": "cbr0",
    "type": "flannel",
    "delegate": {
      "isDefaultGateway": true
    }
  },
  "boot_network": {
    "type": "loopback"
  }
}
```

Some things to note:

* `type: ratchet` is required
* etcd
    - `etcd_host` generally should point to the service we created in the previous step
    - `etcd_port` is the port on which etcd will respond.
    - You can test if `curl etcd-client.ratchet.svc.cluster.local:2379` works and that will let you know if etcd is responding (it'll respond with a 404)
* `child_path` points where the secondary binary for ratchet lives, following these instructions this is the proper path.
* VXLAN
    - `parent_interface` is the interface on which the VXLAN tunnels will reside
    - `parent_address` is the IP address remote VXLANs will use to create a tunnel to this machine.
* `use_labels` should generally be true.
* Alternate CNI plugin
    - `delegate` is a special field. In this we pack in an entire CNI config for another plugin. You'll note that this is set to the exact entry that we have earlier when I show the current config for CNI on one of the minions. When pods are not labeled to use ratchet, they will use this CNI plugin (more on the labeling later).
* `boot_network` -- similar to delegate but when pods are eligble to be processed by Ratchet, they will have an extra interface created with the CNI config as packed into this property. In this case I just set a loopback device, using the loopback CNI plugin.

Great! You've got one all set. But, you need two. So setup another one on a second host.

On my second host I have the same config @ `/etc/cni/net.d/10-ratchet.conf` -- minus one line which differs, and they is the `parent_address` (the `parent_interface` would differ if the nics were named differently on each host), so for example on the second minion I have...

```
[centos@kube-minion-2 ~]$ ip a | grep -iP "(eth0|192)"
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 192.168.1.33/24 brd 192.168.1.255 scope global dynamic eth0

[centos@kube-minion-2 ~]$ cat /etc/cni/net.d/10-ratchet.conf | grep parent
  "parent_interface": "eth0",
  "parent_address": "192.168.1.33",
```

Note that the IP address in `parent_address` matches that of the address on `eth0`.

## Labeling the nodes

Alright, something we're going to want to do is to specify which pods run where for demonstrative purposes. For this we're going to use [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) to tell Kube where to run these pods.

That being said, we will assign a label to each one...

```
[centos@kube-master ~]$ kubectl label nodes kube-minion-1 ratchetside=left
[centos@kube-master ~]$ kubectl label nodes kube-minion-2 ratchetside=right
```

And you can check those labels out if you need to...

```
[centos@kube-master ~]$ kubectl get nodes --show-labels
```

## Running two pods as a baseline test

We are now all configured and ready to rumble with Ratchet. Let's first create a couple pods to make sure everything is running.


---

continue here

ips...

192.168.1.90 master
192.168.1.73 1 
192.168.1.33 2

---

## Running the whole cloud router

## Debugging Ratchet issues


