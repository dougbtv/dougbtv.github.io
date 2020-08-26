---
author: nicklesimba
comments: true
date: 2020-11-27 11:00:00-05:00
layout: post
slug: kubernetes-on-fedora
title: Set up a Kubernetes Cluster on Fedora in 10 Minutes or Less with Kube-Ansible
category: nfvpe
---

Note: credit doug and andrew properly before making a pr!

Kubernetes can sometimes provide an unexpected challenge when trying to go through the installation procedure. If you're like me and just want to play around with Kubernetes without any hassle (raise your hand if you've ever been greeted with an unexpected suite of error messages after running `kubeadm init`), or just want to get an environment up and running quickly on a fedora machine to test something, this guide is for you. Our goal will be to get Kube-Ansible up and running (using Flannel as the CNI plugin), and then create a test pod to make sure everything's working.

## Disclaimers

Before we begin, a couple of things to get out of the way: 

Firstly, we'll be using [Kube-Ansible](https://github.com/redhat-nfvpe/kube-ansible) which is a playbook developed by the RedHat nfvpe group to spin up Kubernetes clusters for development purposes. Generally -- what these playbooks do is bootstrap some hosts for you so they're readied for a Kubernetes install. They then use [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/). If you have more interest in this, follow that previous link to the official docs. The idea of this guide is to provide a step-by-step installation procedure, with common bugs and solutions included. Our goal here is primarily for experimentation. If you're looking for something that's a little bit more production grade, you might want to consider using [OpenShift](https://www.openshift.com/).

Secondly, this guide assumes every step is being ran as the root user, under `/root`. If you are running these commands in user-space, your mileage may vary.

Finally, this guide also assumes usage of Fedora. If you are on a centos machine, feel free to reference this article to find obscure bugfixes, but a better starting point might be [this](https://dougbtv.com/nfvpe/2018/03/21/kubernetes-on-centos/) parent blogpost written by Doug Smith.

## Overview

Through this guide, we'll be walking through the following procedures:

* Install Ansible, clone Kube-Ansible, and configure a few important files to help with the rest of the install.
* Run an ansible script to set up centOS VMs, and run an ansible script to install kubernetes on the VMs.
* Configure the kubernetes ready VMs with CNI plugins as with vanilla Kubernetes.

## Requirements

* A machine with Fedora installed (ideally 30+) that can be used as a host machine
* Git.
* Basic tech [literacy](https://www.youtube.com/watch?v=Q6ctb-Pb3lc)

## Initial Setup

* Change to root user directory
  - `$ cd /root/`
* Install ansible
  - `$ dnf install ansible`
  - This should also include ansible-galaxy to help with installation
* Add the following to .bashrc for later using your favorite text editor
  - `alias ssh-virthost='ssh -o ProxyCommand="ssh -W %h:%p root@localhost"'`
  - Then run `$ source ~/.bashrc`.
  - This allows us to use the "ssh-virthost" alias and will make life easier later. Essentially, it runs the command from our host machine.
* Clone the repo and install requirements
  - `$ git clone https://github.com/redhat-nfvpe/kube-ansible.git && cd kube-ansible`
  - `$ ansible-galaxy install -r requirements.yml`

Now we need to set a loop address for our host machine. This is because the VMs will be communicating with each other through the host, and so we need ssh access to it. Using your favorite text editor, add the following to `inventory/virthost/virthost.inventory`: 
  * `vmhost ansible_host=127.0.0.1 ansible_ssh_user=root ansible_python_interpreter=/usr/bin/python3`

At this point, we've made it through the bulk of setup. Before creating the VMs, we can optionally change default values for them in all.yml, located at `playbooks/ka-init/group_vars/`. The default names for the VMs will be kube-master1, kube-node-1, and kube-node-2. They will be set up with the default roles of master, worker, and worker respectively. We can change VM names as well as roles, or to strictly follow this tutorial, move ahead for now.

## Creating the VMs

We are now done with all set up, and can finally spin up our virtual machines! While installing, there are a few common bugs, and so I've included potential fixes for the ones I ran into in the next section.

* Use a playbook to set up the following VMs: kube-master1, kube-node-1, and kube-node-2. 
  - ansible-playbook -i inventory/virthost/  -e 'network_type=2nics'  -e ssh_proxy_enabled=true playbooks/virthost-setup.yml
* Now use a playbook to install kubernetes on the VMs.
  - ansible-playbook -i inventory/vms.local.generated playbooks/kube-install.yml
  - This may take a bit of time to complete.
* Try connecting to the master VM. 
  - `$ ssh-virthost centos@kube-master1`

## Common Issues and How to Fix Them

There are a few common errors you may encounter while trying to run through the previous section. Hopefully this section should cover most of them and how to fix them.

* If you are getting public key errors, you need to add an ssh key so you can access...yourself! Feel free to manually create any directories that you notice don't have already.
  - `$ cd ~`
  - `$ ssh-keygen` (leave filename and password empty unless you have reason to change them)
  - `$ cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys` (or, if you already have an authorized_keys file, `cat id_rsa.pub` and paste the result into authorized_keys).
  - You should now be able to run the virthost-setup.yml!
* If you are using docker on Fedora, and had to revert to cgroupsv1 do the following	
  - Disable firewalld (It dosn’t play nice with Docker https://github.com/firewalld/firewalld/issues/461) 
    - `$ systemctl stop firewalld`
    - You may want to re-enable after using Kube-Ansible 
  - Ensure sshd is active with `$ systemctl status sshd`. If it's not, start it with `$ systemctl start sshd`.
  - Make sure libvirt’s default virtual network is active
    - List the networks with `$ virsh net-list` 
    - If “Default” is not shown create it with `$ virsh net-start default` 
* If you are getting additional ssh errors, try the following
  - `$ eval `ssh-agent -s``
  - `$ ssh-add ~/.ssh/vmhost/id_vm_rsa`
  - Note: for man in the middle attack warnings, this can happen when trying to reconnect at a later time. Delete the respective key from the known_hosts file as the warning suggests, and then try again.

## Playing with CNI networking

Congratulations! All the difficult set up work is now behind us, and we can move on to configuring CNI plugins! If you've made it this far without running into any bugs, [toss a coin to your glitcher, o valley of silicon](https://youtu.be/PoQJM8SO6V0).

We can first verify that our kubernetes install worked correctly by running `$ kubectl get nodes` and `$ kubectl get pods -A`. You should be able to see the existing pods. Observe that no nodes will be returned at this point. This is expected...because we haven't made any yet! 

Now we will install flannel - feel free to substitute in another CNI plugin such as calico.

* `$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`
  - Now we can run `$ kubectl get nodes` and `$ kubectl get pods -A` and compare the results with the previous ones!

## Clean Up

Because our environment is purely composed of VMs, we can simply exit kube-master-1 and run vm-teardown.yml
  * `$ ansible-playbook -i inventory/virthost/virthost.inventory playbooks/vm-teardown.yml`

Now we have completed a basic CNI configuration set-up/teardown using Kube-Ansible! Thanks for giving it a try. If you'd like to learn more about Kube-Ansible, check out [this](https://github.com/redhat-nfvpe/kube-ansible) link. Additionally, if you'd like to keep experimenting, you can follow the next section to set up Multus-CNI as well.

## Multus Set Up

* Clone multus from Github
  - git clone https://github.com/intel/multus-cni.git
  - You may need to install git first since you’re on a VM.
* Follow the quickstart guide on the multus github under doc/quickstart.md (run the multus-daemonset.yml specifically)
  - Keep in mind that multus requires a CNI such as flannel to be in use first
* Copy from your original device to the kube-master1: net-attach-def.yaml and multus-pod.yaml,
* Then kubectl apply -f net-attach-def.yml and multus-pod.yml
Basic set up done in under 20 steps!

## Closing Notes

Special thanks to Doug Smith for hosting this article, and thank you to Andrew Stoycos for providing solutions to firewalld errors!

Nikhil Simha (me!) - https://github.com/nicklesimba
Doug Smith - https://github.com/dougbtv
Andrew Stoycos - https://github.com/astoycos