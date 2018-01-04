---
author: dougbtv
comments: true
date: 2017-11-30 13:40:02-05:00
layout: post
slug: kubernetes-ipv6
title: Are you exhausted? IPv4 almost is -- let's setup an IPv6 lab for Kubernetes
category: nfvpe
---

It's no secret that there's the inevitability that [IPv4 is becoming exhausted](https://en.wikipedia.org/wiki/IPv4_address_exhaustion). And it's not just tired (ba-dum-ching!). Since we're a bunch of Kubernetes fans, and we're networking fans -- we really want to check out what we can do with IPv6 with Kubernetes. Thanks to some slinky automation by my colleague, [Feng Pan](https://github.com/fepan), contributed to [kube-ansible](https://github.com/redhat-nfvpe/kube-ansible), he was able to implement some creative work by [leblancd](https://github.com/leblancd). In this simple setup today, we're going to deploy Kubernetes with custom binaries from leblancd and have two pods (ideally on different nodes) ping one another with `ping6` and declare victory! In the future let's hope to iterate on what's necessary to get IPv6 functionality in Kubernetes.

There's an ever growing interest in IPv6 for Kubernetes. There's a solid effort by the good folks from the [Kubernetes SIG-Network](https://github.com/kubernetes/community/wiki/SIG-Network). You'll find in the [SIG-Network features spreadsheet](https://docs.google.com/spreadsheets/d/1lHSZBl7YJvKN0qNT8i0oxYaRh38CcemJlUFjD37hhDU/edit#gid=14624465) that IPv6 is slated for the next release. There's probably more to that  Additionally, you can find some more information about the [issues tagged for IPv6 up on the k/k GitHub](https://github.com/kubernetes/kubernetes/pulls?q=is%3Aopen+is%3Apr+label%3Aarea%2Fipv6), too.

There's also a [README for creating an IPv6 lab with kube-ansible](https://github.com/redhat-nfvpe/kube-ansible/blob/master/docs/ipv6.md) on GitHub.

## Limitations

Our goal here with this setup is to make it possible to `ping6` one pod from another. I'm looking forward to using this laboratory to explore the other possibilities and scenarios, however this pod-to-pod `ping6` is the baseline functionality from which to start adventuring into further territory.

## Requirements

TL;DR: A host that can run VMs (or choose your own adventure and bring your baremetal or some other cloud), an editor (anything but Emacs, just kidding), git and Ansible.

To run these playbooks, we assume you have already adventured [warily](https://scryfall.com/card/tsp/171) so far that you have:

* A machine for running Ansible (like your workstation) and [have Ansible installed](http://docs.ansible.com/ansible/latest/intro_installation.html).
* Ansible 2.4 or later (necessary to support `get_url` with IPv6 enabled machines)
* A host capable of running virtual machines, and is running CentOS 7.
* Git. If you don't have git, get git. Don't be a git. We'll clone up in a minute here.

We also disable the "bridged networking" feature we often use and instead uses NAT'ed libvirt virtual machines. 

You may have to disable GRO ([generic receive offload](https://en.wikipedia.org/wiki/Large_receive_offload)) for the NICs on the virtualization host (if you're using one).

An example of doing so is:

```
ethtool -K em3 gro off
```

## Fire up your terminal, and let's clone this repo!

You're going to need to clone up this repo, let's clone at the latest tag that supports this functionality.

```
$ git clone --branch v0.1.6 https://github.com/redhat-nfvpe/kube-ansible.git
```

Cool, enter the dir and surf around if you wish, we'll setup our inventory and necessary variables.

### If you clone master instead of that tag, don't forget to install the galaxy roles!

There's likely some [Ansible Galaxy](https://galaxy.ansible.com/) roles to install, if `find . | grep -i require` shows any files, do a `ansible-galaxy install -r requirements.yml`.

## Inventory and variable setup

Let's look at an inventory and variable overrides to use. Make sure you have a host setup you can run VMs on, that's running CentOS 7, and ensure you can SSH to it.

Here's the initially used inventory, which only really cares about the `virthost`. Here I'm placing this inventory file @ `inventory/my.virthost.inventory`. You'll need to modify the location of the host to match your environment.

```
the_virthost ansible_host=192.168.1.119 ansible_ssh_user=root

[virthost]
the_virthost
```

And the overrides which are based on the examples @ `./inventory/examples/virthost/virthost-ipv6.inventory.yml`. I'm creating this set of extra variables @ `./inventory/extravars.yml` :

```
bridge_networking: false
virtual_machines:
  - name: kube-master
    node_type: master
  - name: kube-node-1
    node_type: nodes
  - name: kube-node-2
    node_type: nodes
  - name: kube-nat64-dns64
    node_type: other
ipv6_enabled: true
```


## Spinning up and access virtual machines

Perform a run of the `virthost-setup.yml` playbook, using the previously mentioned extra variables for override, and an inventory which references the virthost.

```
ansible-playbook -i inventory/my.virthost.inventory -e "@./inventory/extravars.yml" virthost-setup.yml
```

This will produce an inventory file in the local clone of this repo @ `./inventory/vms.local.generated`. And it will also create some SSH keys for you which you'll find in the `.ssh` folder of the user you ran the Ansible playbooks as.

In the case that you're running Ansible from your workstation, and your virthost is another machine, you may need to SSH jump host from the virthost to the virtual machines.

If that is the case, you may add to the bottom of `./inventory/vms.local.generated` a line similar to this (replacing `root@192.168.1.119` with the method you use to access the virtualization host):

```
cat << EOF >> ./inventory/vms.local.generated
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p root@192.168.1.119"'
EOF
```

### Optional: Handy-dandy "ssh to your virtual machines script"

You may wish to log into to the machines in order to debug, or even more likely -- to access the Kubernetes master after an install.

You may wish to create a script, in this example... This script is located at `~/jumphost.sh` and you should change `192.168.1.119` to the hostname or IP address of your `virthost`.

```
# !/bin/bash
ssh -i ~/.ssh/the_virthost/id_vm_rsa -o ProxyCommand="ssh root@192.168.1.119 nc $1 22" centos@$1
```

You would use this script by calling it with `~/jumphost.sh yourhost.local` where the first parameter to the script is the hostname or IP address of the virtual machine you wish to acess.

Here's an example of using it to access the kubernetes master by pulling the IP address from the generated inventory:

```
$ ~/jumphost.sh $(cat inventory/vms.local.generated | grep "kube-master.ansible" | cut -d"=" -f 2)
```

## Deploy a Kubernetes cluster

With the above in place, we can now perform a kube install, and use the locally generated inventory.

```
ansible-playbook -i inventory/vms.local.generated -e "@./inventory/extravars.yml" kube-install.yml
```

SSH into the master, if you created it above, use the handy `jumphost.sh`.

Just double check things are [coming up Milhouse](https://www.youtube.com/watch?v=M67E9mpwBpM) Check out the status of the cluster with `kubectl get nodes` and/or `kubectl cluster-info`.

We'll now create a couple pods via a ReplicationController. Create a YAML resource definition like so:

```
[centos@kube-master ~]$ cat debug.yaml 
apiVersion: v1
kind: ReplicationController
metadata:
  name: debugging
spec:
  replicas: 2
  selector:
    app: debugging
  template:
    metadata:
      name: debugging
      labels:
        app: debugging
    spec:
      containers:
      - name: debugging
        command: ["/bin/bash", "-c", "sleep 2000000000000"]
        image: dougbtv/centos-network-advanced
        ports:
        - containerPort: 80
```

Create the pods with `kubectl` by issuing:

```
$ kubectl create -f debug.yaml
```

Watch 'em come up:

```
[centos@kube-master ~]$ watch -n1 kubectl get pods -o wide
```

## Try it out!

Once those pods are fully running, list them, and take a look at the IP addresses, like so:

```
[centos@kube-master ~]$ kubectl get pods -o wide
NAME              READY     STATUS    RESTARTS   AGE       IP            NODE
debugging-cvbb2   1/1       Running   0          4m        fd00:101::2   kube-node-1
debugging-gw8xt   1/1       Running   0          4m        fd00:102::2   kube-node-2
```

Now you can exec commands in one of them, to ping the other (note that your pod names and IPv6 addresses are likely to differ):

```
[centos@kube-master ~]$ kubectl exec -it debugging-cvbb2 -- /bin/bash -c 'ping6 -c5 fd00:102::2'
PING fd00:102::2(fd00:102::2) 56 data bytes
64 bytes from fd00:102::2: icmp_seq=1 ttl=62 time=0.845 ms
64 bytes from fd00:102::2: icmp_seq=2 ttl=62 time=0.508 ms
64 bytes from fd00:102::2: icmp_seq=3 ttl=62 time=0.562 ms
64 bytes from fd00:102::2: icmp_seq=4 ttl=62 time=0.357 ms
64 bytes from fd00:102::2: icmp_seq=5 ttl=62 time=0.555 ms
```

Finally pat yourself on the back and enjoy some IPv6 goodness.