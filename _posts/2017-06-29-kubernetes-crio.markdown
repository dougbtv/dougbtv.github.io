---
author: dougbtv
comments: true
date: 2017-06-29 08:05:03-05:00
layout: post
slug: kubernetes-crio
title: Look ma, No Docker! Kubernetes with CRI-O, and no Docker at all!
category: nfvpe
---

This isn't just a stunt like riding a bike with no hands -- it's probably the future of how we'll use Kubernetes. Today, we're going to spin up Kubernetes using [cri-o](http://cri-o.io/) which uses the Kubernetes container runtime interface with [OCI](https://www.opencontainers.org/) (open containers initive) compatible runtimes. That's a mouthful, but, the gist is -- it's a way to use Kubernetes without Docker! That's what we'll do today. And to add a cherry on top, we're also going to build a container image without Docker, too. We won't go in depth on images today -- our goal will be to get a Kubernetes up without Docker, with cri-o, and we'll run a pod on it to prove it out. 

We're not going to have much luck with building and managing images. In a coming eposide we'll add [Buildah](https://github.com/projectatomic/buildah) into the mix, a project out of Project Atomic which can build OCI images. Then we can expand to having a whole workflow without Docker. But today, I promise that you won't do a single `docker {run,build,ps}`, not a one.

I saw [this tweet](https://twitter.com/soltysh/status/870534766914920448) from [@soltysh](https://twitter.com/soltysh) on Twitter which linked me to the [cri-o ansible](https://github.com/cri-o/cri-o-ansible) playbook which inspired me to implement the same concept in my [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible) playbooks. Inspired is the wrong word -- more like made me ultra giddy to give it a try. 

Here's the thing, editorially -- I love Docker^hMoby, and I am a firm believer that what Docker did was change the landscape for how we manage and deploy applications. But, it's not wise to have a majority rule of the technology we use. So, I'm really excited for CRI-O. This is a game changer for the whole landscape, and I think the open governance model of CRI-O will be a huge boon for all parties involved (including Docker, too).

You might enjoy enjoy the infamous [Kelsey Hightower's cri-o-tutorial](https://github.com/kelseyhightower/cri-o-tutorial).

## Requirements

We're going to use [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible) -- and this will spin up virtual machines for you if you want. If you don't want -- you could setup physical machines with CentOS 7, and skip on to the part where you modify the inventory for that. We'll basically start from square one here and setup a virtual machine host for you, but, it's up to you if you want that. Should you go with the virt-host method, you'll need to strap that machine with CentOS 7, and give yourself some SSH keys.

So in short... The main consideration here is to have a machine you can deploy to (which could in theory, be your local machine, it might work with Fedora, and will certainly work with CentOS) -- and you'll need to have [Ansible installed](http://docs.ansible.com/ansible/intro_installation.html) on a machine that can access the machine(s) with SSH. 

## What's the hard part?

Honestly, most of this is really easy. The hardest part is managing your inventories and running my playbooks if you're unfamiliar with them. I'll give a recap here of how to do that. 

We're using my [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible) playbooks, and if you aren't familiar with them, I recommend you check out my intro [blog article on how to install Kubernetes](http://dougbtv.com//nfvpe/2017/02/16/kubernetes-1.5-centos/) which goes in depth on these playbooks -- I take them for granted sometimes and that will be useful as a reference if I miss something that I took as obvious.

## Virtual machine host & spinning up the virtual machines

As I mentioned previously -- skip this section if you already have machine provisioned. Otherwise, get yourself a fresh (or existing should likely be ok) CentOS 7 install where we can run VMs -- so physical is preferable unless, yo dawg, I heard you like nested virtualization.

Alright, first thing's first, let's clone the kube-centos-ansible playbooks. 

(note: you're cloning at a specific tag	to reference an	old style inventory, if	you wish you can remove	the `--branch` parameter, and go via head, and figure out the new inventory, just browse the `./inventory` dir)

```
$ git clone https://github.com/redhat-nfvpe/kube-centos-ansible.git && cd kube-centos-ansible
```

In there I'm going to have an inventory you should modify, so go ahead and modify this and put in the proper hostname/ip.

```
cat ./inventory/virthost.inventory 
kubehost ansible_host=192.168.1.119 ansible_ssh_user=root

[kubehost]
kubehost
```

Now that you have that, you should be able to run the `virt-host-setup.yml` playbook. Note that we're specifying 4 gigs of RAM for each virtual machine. GCC was not super happy with just 2 gigs when going to compile CRI-O, so I decided to bump it up a bit (imagine calling 2 gigs of RAM "a bit" in 1995? That would be funny).

```
$ ansible-playbook -i inventory/virthost.inventory -e "ram_mb=4096"  virt-host-setup.yml
```

Importantly, this will create some ssh keys on that target virtual machine host that you'll want to put on the machine where you're running Ansible.

```
[root@your-virt-host ~]# ls ~/.ssh/id_vm_rsa
```

Also! It will show you a list of IP addresses for the machines you created. We use those in the next step.

You'll also note at this point there are virtual machines running, you can see them with `virsh list --all`.

## Readying the inventory of your virtual machines

Alright, now let's modify the VM inventory. So go ahead and modify the `./inventory/vms.inventory`

Main things here are:

1. Modify the hosts at the top to match the IPs of the machines you just provisioned
2. Modify the jump host information, e.g. for the virtual machine host. (skip this step if you brought your own hosts)

These are the two lines you really care about for step 2.

```
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p root@192.168.1.119"'
ansible_ssh_private_key_file=/home/doug/.ssh/id_vm_rsa
```

Change the IP to the IP of your virtual machine host, and set the private key location to where you are keeping the private key on your local machine -- e.g. the one that was created for you on the virtual machine host (that is... scp it to your local machine and then reference it here)

## Let's run this playbook!

So there's a bit more setup than what's the meat and potatoes... We're about to do that now. (Note that we specify a kubernetes version to install here, there's a [current issue](https://github.com/redhat-nfvpe/kube-centos-ansible/issues/37) with the playbooks that doesn't yet know how to handle that.)

```
$ ansible-playbook -i inventory/vms.inventory -e 'container_runtime=crio'  kube-install.yml
```

## Verify the installation

Ok cool... So let's do the first bit of verification... That's there's no, and I mean NO DOCKER. Aww. Yes.

Log yourself into the master (and minions, I know you're incredulous, so go for it).

```
[centos@kube-master ~]$ sudo docker -v
sudo: docker: command not found
```

Just how we like it!

Now... List that you have some connected nodes.

```
[centos@kube-master ~]$ kubectl get nodes
NAME            STATUS    AGE       VERSION
kube-master     Ready     4m        v1.6.6
kube-minion-1   Ready     3m        v1.6.6
```

Ok, that's all well and good... but, is anything running?

Should be!

```
[centos@kube-master ~]$ kubectl get pods --all-namespaces
```

Word!

## Running a pod

Let's use my favorite little nginx example.. Go ahead and put this yaml into a file named `nginx.yaml`:

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

Now go ahead and create using that, a la:

```
[centos@kube-master ~]$ kubectl create -f nginx.yaml 
```

And watch the two pods come up...

```
[centos@kube-master ~]$ watch -n1 kubectl get pods
```

Cool, now let's see if we can reach an nginx...

```
[centos@kube-master ~]$ curl -s $(kubectl describe pod $(kubectl get pods | grep nginx | head -n 1 | awk '{print $1}') | grep "^IP" | awk '{print $2}') | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

And there it is! Mission complete.

## Some commands to get you around

So -- you don't have docker, and there's some regular ole things you'd like to do. 

So how about the running processes? You can use `runc` for this, such as:

```
[centos@kube-master ~]$ sudo runc list
```

And get some help for it, to see some other things running:

```
[centos@kube-master ~]$ sudo runc --help
```

## Some of my show stoppers.

One of the first things I ran into was that `kubeadm` was complaining I didn't have docker -- well, I know that `kubeadm` ;) So, I tried to skip preflight checks...

```
kubeadm init --skip-preflight-checks --pod-network-cidr 10.244.0.0/16
```

And that appeared to have worked. I think I saw something zip by on the kubernetes slack channels about this, maybe even in the `kubeadm` channel.

I talked with the awesome folks in the `#cri-o` channel on freenode, and they noted that this is a known issue with kubeadm and they've got PR's open so that kubeadm knows it's OK to use another runtime. Awesome!
