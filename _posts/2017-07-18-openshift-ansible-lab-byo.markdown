---
author: dougbtv
comments: true
date: 2017-07-18 14:00:02-05:00
layout: post
slug: openshift-ansible-lab-byo
title: BYOB - Bring your own boxen to an OpenShift Origin lab!
category: nfvpe
---

Let's spin up a [OpenShift Origin](https://www.openshift.org/) lab today, we'll be using [openshift-ansible](https://github.com/openshift/openshift-ansible) with a "BYO" (bring your own) inventory. Or I'd rather say "BYOB" for "Bring your own [boxen](https://en.wiktionary.org/wiki/boxen)". OpenShift Origin is the upstream OpenShift -- in short, OpenShift is a PaaS (platform-as-a-service), but one that is built with a distribution of Kubernetes, and in my opinion -- is so valuable because of its strong opinions, which guide you towards some best practices for using Kubernetes for the enterprise. In addition, we'll use my [openshift-ansible-bootstrap](https://github.com/dougbtv/openshift-ansible-bootstrap) which we can use to A. spin up some VMs to use in the lab, and/or B. Setup some basics on the host to make sure we can properly install OpenShift Origin. Our goal today will be to setup an OpenShift Origin cluster with a master and two compute nodes, we'll verify that it's healthy -- and we'll deploy a very basic pod.

If you're itching to get your hands on the keyboard, skip down to "Clone Doug's `openshift-ansible-bootstrap`" to omit the intro.

## What, exactly, are we going to deploy?

The gist is we're going to use Ansible from "some device" (in my case, my workstation, and I'd guess yours, too). We'll then provision a machine to be a "virt-host" -- a host for running virtual machines. Then we'll spin up 3 virtual machines (with libvirt) to run OpenShift on. Those virtual machines are connected to a `br0` bridge which will allow these virtual machines to have IP addressing on your LAN. (As opposed to say, a NAT'ed IP address)

![architecture diagram](http://i.imgur.com/btOnMEK.png)

## Requirements

In this setup we use a CentOS 7 virtual machine host, you'll need decent size on it. You might be able to trim down some of these, but, what I'm using is a baremetal node with 16 cores, using 4 cores per VM, 96 gigs of RAM, and I have 1TB spinning disk. 

You'll need at least:

* 48 gigs of RAM (16 per VM)
* ~240 gigs of HDD (~80 gigs per VM)
* 6-8 cores (2 core per VM, I recommend 4 per VM)

This walk-through assumes that you have a host (like that) with CentOS 7.3 up and running (and hopefully you have some updated packaged and a late kernel, too).

You'll need a host from which to run Ansible, and you'll need Ansible installed. Additionally, we're going to be using [OpenShift-Ansible](https://github.com/openshift/openshift-ansible) which requires Ansible 2.2.2.0 or greater. This could be the same as your virtual host. Make sure you have SSH keys to your target box.

Additionally -- while I use a VM lab, you could definitely spin up baremetal, or some VMs on "the cloud platform of your choosing" (and I hope for your sake, you don't use one that has vendor lock-in). Just read through and skip the VM provisioning portion.

## Limitations

Really -- you'll want a DNS server for your cluster if you're doing anything bigger than this, and even this setup could benefit from a DNS implementation. I don't really go there in this implementation.

There is no HA components herein. Those may be extended to this lab environment when the right use-case for the lab comes along.

Additionally, since we're using a single master node, there won't be an official load balancer. The load balancer conflicts with some master service, and required a node dedicated to it. (Although, in theory you can probably schedule pods on that node, too.)

## Docker storage driver

One of the bumps in the road I ran into while I was working on this was the Docker storage driver.

OpenShift does some great things for us, and that [OpenShift-Ansible](https://github.com/openshift/openshift-ansible) honors -- one of those things being that it discourages you from using a loopback storage driver. 

I followed the instructions for [configuring `direct-lvm` storage for Docker](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#configure-direct-lvm-mode-for-production) from the Docker documentation. 

Mostly though, these are covered in the playbooks, so, if you want, dig into those to see how I sorted it out. It's worth noting that in the most recent Docker versions (the version used here at the time of writing is 1.12.x) make setting up the `direct-lvm` volumes much easier, and it does all volume actions automagically. In short, what I do is dedicate a disk to each VM and then tell Docker to use it.

## Clone Doug's `openshift-ansible-bootstrap`

I'll assume now that you've got a machine to use that we can spin up virtual machines on, and that you have SSH keys from whatever box you're going to run ansible on to that host.

I've got a few playbooks put together in a repo that'll help you gets some basics on a few hosts to use for spinning up OpenShift Origin with a BYO inventory. I call it, boringly, [openshift-ansible-bootstrap](https://github.com/dougbtv/openshift-ansible-bootstrap).

Go ahead and clone that.

```
$ git clone https://github.com/dougbtv/openshift-ansible-bootstrap.git
```

## Setup the virtual machine host.

Alright, first thing let's open up the `./inventory/inventory` file in the clone. Modify the `virt_host` line (in the first few lines) to have a `ansible_host` that has the IP (or hostname) of the machine we're going to provision.

```
# Setup this host first, and put the IP here.
virt_host ansible_host=192.168.1.42 ansible_ssh_user=root
```

You'll need to specify the NIC that you use to access the LAN/WAN on that host with:

```
bridge_physical_nic=enp1s0f1
```

(e.g. replace `enp1s0f1` with `eth0` if that's what you have.)

Additionally (in order for the playbook to discover the IP address of the VMs it creates), you'll need to specify the CIDR for the network on which that NIC operates...

```
bridge_network_cidr=192.168.1.0/24
```

Now that you have that setup, we can run the `virt-host-setup.yml`, like so:

```
$ ansible-playbook -i inventory/inventory virt-host-setup.yml
```

Oh is it coffee time? IT IS COFFEE TIME. Fill up a big mug, and I recommend stocking on up [Vermont Coffee Company's Tres](http://www.vermontcoffeecompany.com/Vermont_Coffee_Company/Viva_Cafe_Dominicano_story.html). It's legit.

In this process we have:

* Installed dependencies to run VMs with libvirt
* Spun up 3 VMs (and pick up their IP addresses)

## Setup the inventory for the virtual machines (and grab the ssh keys)

Look in the output from the playbook and look for a section called: "Here are the IPs of the VMs", grab those IPs and add them into the `./inventory/inventory` file in this section:

```
# After running the virt-host-setup, then change these to match.
openshift-master ansible_host=192.168.1.183
openshift-minion-1 ansible_host=192.168.1.130
openshift-minion-2 ansible_host=192.168.1.224
```

Ok, but, that's no good without grabbing the SSH key to access these. You'll find the key to them on the virt host, in root's directory, the file should be here:

```
$ cat /root/.ssh/id_vm_rsa
```

Take that file and put it on your ansible machine, and we'll also add that into the inventory.

Find this section in the inventory, and modify it to match where you put the file (keep the `ansible_ssh_user` the same, in most cases)

```
[openshiftnodes:vars]
ansible_ssh_user=centos
ansible_ssh_private_key_file=/home/doug/.ssh/id_openshift_hosts
```

## Modify the virtual machine hosts to get ready for an OpenShift Ansible run.

Cool -- now go ahead and run the `bootstrap.yml` playbook which will setup these VMs to be readied for an openshift Ansible install.

```
$ ansible-playbook -i inventory/inventory bootstrap.yml
```

There's a few things this does that really helps us out so that openshift-ansible can do the magic we need it to do.

* It installs the correct docker version, and sets `direct-lvm` storage for Docker
* It sets up the host files on the machines so that we don't need DNS

That one should finish in a pretty reasonable amount of time.

## Start the OpenShift Ansible run.

In the `openshift-ansible-bootstrap` clone's root, you'll find a file `final.inventory` which is the inventory we're going to use for openshift-ansible -- except again, we'll have to replace the IPs in the first three lines of that file. (These will match what you created in the last step for the `bootstrap.yml`)

Here's the whole thing in case you need it:

```
openshift-master ansible_host=192.168.1.183
openshift-minion-1 ansible_host=192.168.1.130
openshift-minion-2 ansible_host=192.168.1.224

[OSEv3:children]
masters
nodes
etcd
# lb
# nfs

[OSEv3:vars]
ansible_ssh_user=centos
ansible_become=yes
debug_level=2
openshift_deployment_type=origin
# openshift_release=v3.6
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
ansible_ssh_private_key_file=/root/.ssh/id_vm_rsa

[masters]
openshift-master

[etcd]
openshift-master

# [lb]
# openshift-master

[nodes]
# make them unschedulable by adding openshift_schedulable=False any node that's also a master.
openshift-master openshift_node_labels="{'region': 'infra', 'zone': 'default'}" openshift_schedulable=true
openshift-minion-[1:2] openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
```

Alright, now, let's ssh into the virtual machine host, and we'll find that it's cloned the openshift-ansible repo. 

So move into that directory...

```
$ cd /root/openshift-ansible/
```

And put the contents of that final inventory into `./my.inventory`

## Drum roll please, begin the openshift-ansible run...

Now you can run the openshift ansible playbook like so:

```
$ ansible-playbook -i my.inventory ./playbooks/byo/config.yml
```

Now, make 10 coffees -- and/or wait for your Vermont Coffee Company order to complete and then brew that coffee. This takes a bit.

## Verifying the setup.

So, we'll assume that openshift-ansible completed without a hitch (and if it didn't? Give a read-through of the error, and give a shot at fixing it, and with that info in hand open up an issue or PR on my bootstrap playbooks). Now, we can look at the node status.

SSH into the master, and run:

```
[centos@openshift-master ~]$ oc status
[...snip...]
[centos@openshift-master ~]$ oc get nodes
NAME                               STATUS    AGE
openshift-master.example.local     Ready     52m
openshift-minion-1.example.local   Ready     52m
openshift-minion-2.example.local   Ready     52m
```

You should have 3 nodes, and you might have noticed something in the `./final.inventory` -- I've told OpenShift that it's OK to schedule pods on the master. We're using a lot of resources for this lab, so, might as well make use of the master, too.

If you want to, set a hosts file on your workstation to point `openshift-master.example.local` at the IP we've been using as the inventory IP address. And then point a browser @ https://openshift-master.example.local:8443/ and accept the certs to kick up the dashboard.

## Let's kick off a pod.

Alright, why don't we use my all time handy favorite nginx pod!

First, let's create a new project.

```
[centos@openshift-master ~]$ oc new-project sample
```

We're going to use a public nginx container image, so, this one assumes it can run as the user it choses, so... We're going to allow this. In your own production setup, you'll likely massage [the users and SCCs](https://blog.openshift.com/understanding-service-accounts-sccs/) to fit a cleaner mold.

So in this case, we'll add the `anyuid` SCC to the default user.

```
[centos@openshift-master ~]$ oc adm policy add-scc-to-user anyuid -z default
```

Then, create a `nginx.yaml` with these contents:

```
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
```

Create the replica set we're defining with:

```
[centos@openshift-master ~]$ oc create -f nginx.yaml 
```

Watch the pods come up...

```
[centos@openshift-master ~]$ watch -n1 oc get pods
```

Should the pod fail to come up, do a `oc describe pod nginx-A1B2C3` (replacing the pod name with the one from `oc get pods`)

Then... We can curl something from it. Here's a shortcut to get you one of the pod's IP addresses and curl it.

```
[centos@openshift-master ~]$ curl -s $(oc describe pod $(oc get pods | tail -n1 | awk '{print $1}') | grep -P "^IP" | awk '{print $2}') | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

And there you have it!