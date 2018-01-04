---
author: dougbtv
comments: true
date: 2017-08-17 14:30:01-05:00
layout: post
slug: ratchet-vxlan
title: Ratchet CNI -- Using VXLAN for network isolation for pods in Kubernetes
category: nfvpe
---

In today's episode we're looking at [Ratchet CNI](https://github.com/dougbtv/ratchet-cni), an implementation of [Koko](https://github.com/redhat-nfvpe/koko) -- but in CNI, the container networking interface that is used by Kubernetes for creating network interfaces. The idea being that the network interface creation can be performed by Kubernetes via CNI. Specifically we're going to create some network isolation of network links between containers to demonstrate a series of "cloud routers". We can use the capabilities of Koko to both create vEth connections between containers when they're local to the same host, and then VXLAN tunnels to containers when they're across hosts. Our goal today will be to install & configure Ratchet CNI on an existing cluster, we'll verify it's working, and then we'll install a cloud router setup based on [zebra pen](https://github.com/dougbtv/zebra-pen) (a cloud router demo).

Here's what the setup will look like when we're done:

![diagram](http://i.imgur.com/rpO2A20.png)

The gist is that the green boxes are Kubernetes minions which run pods, and the blue boxes are pods running on those hosts, and the yellow boxes are the network interfaces that will be created by Ratchet (and therefore Koko). In this scenario, just one VXLAN tunnel is created when going between the hosts.

So that means we'll route traffic from "CentOS A" container, across 2 routers (which use OSPF) to finally land at "CentOS B", and have a ping come back across the links.

Note that Ratchet is still a prototype, and some of the constraints of it are limited to the static way in which interfaces and addressing is specified. This is indeed a limitation, but is intended to illustrate how you might specify the links between these containers.

## Requirements

Required:

* A Kube cluster, [spin one up my way](https://github.com/redhat-nfvpe/kube-ansible) if you wish.
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

Next, let's assess what you have available for networking. Mine is pretty simple. Each of my nodes have a single nic -- `eth0`, and it's on the `192.168.1.0/24` network, and that network is essentially flat -- it can access the WAN over that NIC, and also the other nodes on the network. Naturally, in real life -- your network will be more complex. But, in this step... Choose the proper NIC and IP address for your setup.

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

Let's create these pods using this yaml:

```
---
apiVersion: v1
kind: Pod
metadata:
  name: primary-pod
  labels:
    app: primary-pod
    ratchet: "true"
    ratchet.pod_name: "primary-pod"
    ratchet.target_pod: "primary-pod"
    ratchet.target_container: "primary-pod"
    ratchet.public_ip: "1.1.1.1"
    ratchet.local_ip: "192.168.2.100"
    ratchet.local_ifname: "in1"
    ratchet.pair_name: "pair-pod"
    ratchet.pair_ip: "192.168.2.101"
    ratchet.pair_ifname: "in2"
    ratchet.primary: "true"
spec:
  containers:
    - name: primary-pod
      image: dougbtv/centos-network
      command: ["/bin/bash"]
      args: ["-c", "while true; do sleep 10; done"]
  nodeSelector:
    ratchetside: left
---
apiVersion: v1
kind: Pod
metadata:
  name: pair-pod
  labels:
    app: pair-pod
    ratchet: "true"
    ratchet.pod_name: pair-pod
    ratchet.primary: "false"
spec:
  containers:
    - name: pair-pod
      image: dougbtv/centos-network
      command: ["/bin/bash"]
      args: ["-c", "while true; do sleep 10; done"]
  nodeSelector:
    ratchetside: right
```

Likely the most important things to look at are these labels:

```
ratchet: "true"
ratchet.pod_name: "primary-pod"
ratchet.target_pod: "primary-pod"
ratchet.target_container: "primary-pod"
ratchet.local_ip: "192.168.2.100"
ratchet.local_ifname: "in1"
ratchet.pair_name: "pair-pod"
ratchet.pair_ip: "192.168.2.101"
ratchet.pair_ifname: "in2"
ratchet.primary: "true"
```

These are how ratchet knows how to setup the interfaces on the pods. You set up each pod as pairs. Where there's a "primary" and a "pair". You need to (as of now) know the name of the pod that's going to be the pair. Then you can set the names of the interfaces, and which IPs are assigned. In this case we're going to have an interface called `in1` on the primary side, and an interface named `in2` on the pair side. The primary will be assigned the IP address `192.168.2.100` and the pair will have the IP address `192.168.2.101`. 

Of all of the parameters, the keystone is the `ratchet: "true"` parameter, which tells us that ratchet should process this pod -- otherwise, it will pass through the pod to another CNI plugin given the `delegate` parameter in the ratchet configuration.

I put that into a file `example.yaml` and created it as such:

```
[centos@kube-master ~]$ kubectl create -f example.yaml 
```

And then watched it come up with `watch -n1 kubectl get pods`. Once it's up, we can check out some stuff.

But -- you should also check out which nodes they're running on to make sure you got the labelling and the nodeSelector's correct. You can do this by checking out the description of the pods, and looking for the node values.

```
$ kubectl describe pod primary-pod | grep "^Node"
$ kubectl describe pod pair-pod | grep "^Node"
```

Now that you know they're on differnt nodes, let's enter the primary pod. 

```
[centos@kube-master ~]$ kubectl exec -it primary-pod -- /bin/bash
```

Now we can take a look at the interfaces...

```
[root@primary-pod /]# ip a | grep -P "(^\d|inet\s)"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    inet 127.0.0.1/8 scope host lo
7: in1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN qlen 1000
    inet 192.168.2.100/24 brd 192.168.2.255 scope global in1
```

Note that there's two interfaces:

* `lo` which is a loopback created by the `boot_network` CNI pass through parameter in our configuration.
* `in1` which is a vxlan, assigned the `192.168.2.100` IP address as we defined in the pod labels.

Let's look at the vxlan properties like so:

```
[root@primary-pod /]# ip -d link show in1
7: in1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether 9e:f4:ab:a0:86:7a brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 0 
    vxlan id 11 remote 192.168.1.33 dev 2 srcport 0 0 dstport 4789 l2miss l3miss ageing 300 addrgenmode eui64 
```

You can see that it's a vxlan with an id of 11, and the remote side is @ `192.168.1.33` which is the IP address of the second minion node. That's looking correct. 

That being said, we can ping the other side now, that we know is @ IP address of `192.168.2.101`

```
[root@primary-pod /]# ping -c1 192.168.2.101
PING 192.168.2.101 (192.168.2.101) 56(84) bytes of data.
64 bytes from 192.168.2.101: icmp_seq=1 ttl=64 time=0.546 ms
```

Excellent! All is well and good, let's destroy this pod, and shortly we'll move onto the more interesting setup.

```
[centos@kube-master ~]$ kubectl delete -f example.yaml 
```

## Quick clean-up procedure

Ratchet is in need of some clean-up routines of its own, and since they're not implemented yet, we have to clean up the etcd data ourselves. So let's do that right now.

We're going to create a kubernetes job to delete, with this yaml:

```
---
apiVersion: batch/v1
kind: Job
metadata:
  name: etcd-delete
spec:
  template:
    metadata:
      name: etcd-delete
    spec:
      containers:
      - name: etcd-delete
        image: centos:centos7
        command: ["/bin/bash"]
        args:
          - "-c"
          - >
            ETCD_HOST=etcd-client.ratchet.svc.cluster.local;
            curl -s -L -X DELETE http://$ETCD_HOST:2379/v2/keys/ratchet\?recursive=true;
      restartPolicy: Never
```

I created this file as `job-delete-etcd.yaml`, and then executed it as such:

```
[centos@kube-master ~]$ kubectl create -f job-delete-etcd.yaml 
```

And I want to watch it come to completion with:

```
[centos@kube-master ~]$ watch -n1 kubectl get pods --show-all
```

You can now remove the job if you wish:

```
[centos@kube-master ~]$ kubectl delete -f job-delete-etcd.yaml 
```

## Running the whole cloud router

Next, we're going to run a more interesting setup. I've got [the YAML resource definitions stored in this gist](https://gist.github.com/dougbtv/d5db4db7dde33349df8b25d8998bfeb7), so you can peruse them more deeply. 

A current limitation is that there are 2 parts, you have to run the first part, wait for the pods to come up, then you can run the second part. This is due to the fact that the current VXLAN implementation of Ratchet is a sketch, and doesn't take into account a few different use cases -- one of which being that there is sometimes more than "just a pair" -- and in this case, there's 3 pairs and some overlap. So we create them in an ordered fashion to let Ratchet think of them just as pairs -- because otherwise if we create them all right now, we get a race condition, and usually the vEth wins, so... We're working around that here ;)

Let's download those yaml files.

```
$ curl -L https://goo.gl/QLGB2C > cloud-router-part1.yaml
$ curl -L https://goo.gl/aQotzQ > cloud-router-part2.yaml
```

Now, create the first part, and let the pods come up.

```
[centos@kube-master ~]$ kubectl create -f cloud-router-part1.yaml 
[centos@kube-master ~]$ watch -n1 kubectl get pods --show-all
```

Then you can create the second part, and watch the last single pod come up.

```
[centos@kube-master ~]$ kubectl create -f cloud-router-part2.yaml 
[centos@kube-master ~]$ watch -n1 kubectl get pods --show-all
```

Using the diagram up at the top of the post, we can figure out that the "Centos A" box routes through both quagga-a and quagga-b before reaching Centos B -- so that means if we ping Centos B from Centos A -- that's an end-to-end test. So let's run that ping:

```
[centos@kube-master ~]$ kubectl exec -it centosa -- /bin/bash
[root@centosa /]# ping -c5 192.168.4.101
PING 192.168.4.101 (192.168.4.101) 56(84) bytes of data.
64 bytes from 192.168.4.101: icmp_seq=1 ttl=62 time=0.399 ms
[... snip ...]
```

Hurray! Feel free to go and dig through the rest of the pods and check out `ip a` and `ip -d link show` etc. Also feel free to enter the quagga pods and run `vtysh` and see what's going on in the routers, too.

## Debugging Ratchet issues

This is the very short version, but, there's basically two places you want to look to see what's going on.

* `journalctl -u kubelet -f` will give you the output from ratchet when it's run by CNI proper, this is how it's initially run.
* `tail -f /tmp/ratchet-child.log` -- this is the log from the child process, and likely will give you the most information. Note that this method of logging to temp is an ulllllltra hack. And I mean it's a super hack. It's just a work-around to get some output while debugging for me.




