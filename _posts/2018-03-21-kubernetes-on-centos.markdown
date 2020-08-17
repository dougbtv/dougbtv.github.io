---
author: dougbtv
comments: true
date: 2018-03-21 14:20:01-05:00
layout: post
slug: kubernetes-on-centos
title: Spin up a Kubernetes cluster on CentOS, a choose-your-own-adventure 
category: nfvpe
---

So you want to install Kubernetes on CentOS? Awesome, I've got a little choose-your-own-adventure here for you. If you choose to continue installing Kubernetes, keep reading. If you choose to not install Kubernetes, skip to the very bottom of the article. I've got just the recipe for you to brew it up. It's been a year since [my last article on installing Kubernetes](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) on CentOS, and while it's still probably useful -- some of the Ansible playbooks we were using have changed significantly. Today we'll use [kube-ansible](https://github.com/redhat-nfvpe/kube-ansible) which is a playbook developed by my team and I to spin up Kubernetes clusters for development purposes. Our goal will be to get Kubernetes up (and we'll use Flannel as the CNI plugin), and then spin up a test pod to make sure everything's working [swimmingly](https://www.youtube.com/watch?v=e8yx4k4tzqE).

## What's inside?

Our goal here is to spin up a development cluster of Kubernetes machines to experiment here. If you're looking for something that's a little bit more production grade, you might want to consider using [OpenShift](https://www.openshift.com/) -- the bottom line is that it's a lot more opinionated, and will guide you to make some good decisions for production, especially in terms of reliability and maintenance. What we'll spin up here is more-or-less the bleeding edge of Kubernetes. This project is more appropriate for infrastructure experimentation, and is generally a bit more fragile. 

We'll be using Ansible -- but you don't have to be an Ansible expert. If you can get it installed (which should be as easy as a `pip install` or `dnf install`) -- you're well on your way. I'll give you the command-by-command rundown here, and I'll provide example inventories (which tell Ansible which machines to operate on). We use [kube-ansible](https://github.com/redhat-nfvpe/kube-ansible) extensively here to do the job for us.

Generally -- what these playbooks do is bootstrap some hosts for you so they're readied for a Kubernetes install. They then use [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/). If you have more interest in this, follow that previous link to the official docs, or check out my (now likely a bit dated) article on [manually installing Kubernetes on CentOS](http://dougbtv.com/nfvpe/2017/02/09/kubernetes-on-centos/).

Then, post install, the playbooks can install some [CNI plugins](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/) -- the plugins that Kubernetes uses to configure the networking on the cluster. By default we spin up the cluster with Flannel. 

## Brief overview of the adventure.

So what exactly are we going to do?

* You'll clone a repo to help install Kube on CentOS.
* You'll make a choice:
    - To provision a CentOS host to use as a virtual machine host which hosts the virtual guests which will comprise your cluster
    - Install CentOS on any number of machines (2+ recommended) which will become the nodes which comprise your cluster.
* Install Kubernetes
* Verify the installation by running a couple pods.

## Requirements

Overall you're required to have:

* Some box with [Ansible installed](http://docs.ansible.com/ansible/latest/intro_installation.html) -- you don't need to be an Ansible expert.
* Git. 
* You guessed it, a coffee in hand. Beans must have been ground at approximately the time of brewing, and your coffee was poured from 12" or higher into your drinking vessel to help aerate the coffee. Seeing it's a choose your own adventure -- you may also choose tea.You'll just be suffering a little. But, grab some [Smith Teamaker's Rooibos](https://www.smithtea.com/collections/herbal-infusions/products/rooibos), it's pretty fine.

Secondarily, there's a choose-your-own-adventure part. Basically, you can choose to either: 

1. Provision a host that can run virtual machines, or
2. Spin up whatever CentOS hosts yourself.

Generally -- I'd suggest #2. Hopefully you have a way to spin up hosts in your own environment. You could use anything from [spacewalk](https://spacewalkproject.github.io/), to [bifrost](https://docs.openstack.org/bifrost/latest/), or... If you're hipster cool, maybe you're even using [matchbox](https://coreos.com/matchbox/docs/latest/matchbox.html). 

Mostly the playbooks used to spin up virtual machines for you herein are for my own quick iteration when I'm quickly building (and destroying) clusters, and trying different setups, configurations, new features, CNI plugins, etc. Feel free to use it, but, it could just slow you down if you otherwise have a workflow for spinning up boxen. *Sidenote: For years I called a virtualization host I was using in a development environment "deathstar" because the rebels kept destroying the damn thing. Side-sidenote: I was a rebel.*

If you've choosen *"1. Provision a host that can run virtual machines."* -- then you're just required to have a host that can run virtual machines. I assume there's already a CentOS operating system on it. You should have approximately 60-120+ gigs of disk space free, and maybe 16-32 gigs of RAM. That should be more than enough.

If you chose the adventure *"2. Spin up whatever CentOS hosts yourself."* -- then go ahead and spin those CentOS machines up yourself, and I'd recommend 3 of them. 2 is fine too. 1 will just not be nearly as much fun. Generally, I'd recommend 4 gig of RAM a piece, and maybe 20+ gig free for each node.

I admit that the box sizing recommendations are fairly arbitrary. You'd likely size them according to your workloads, but, these are essentially "medium range guesses" to make sure it works.

## Clone the kube-ansible repo.

Should be fairly simple, just clone 'er right up:

```
$ git clone -b v0.5.0 https://github.com/redhat-nfvpe/kube-ansible.git && cd kube-ansible
```

You'll note that we're cloning at a particular tag -- `v0.5.0`. If you want, omit the `-b v0.5.0`, which will make it so you're on the master branch. In theory, it should be fine. I chose a particular tag for this article so it'll still be relevant in the case that we (inevitably) make changes to the kube-ansible repo.

It'll change directory into that directory with the copy-and-pasted command, and then you can initialize the included roles...

```
$ ansible-galaxy install -r requirements.yml
```

You'll note here that we're cloning at a particular tag so that things don't change and I can base the documentation on it. If you're feeling particularly, ahem, adventurous -- you can choose the adventure to remove the `-b 0.2.1` parameter, and clone at master HEAD. I'm hopeful that there's some maturity on these playbooks and that shouldn't matter much, but, at least at this tag it'll match your experience with this article. Granted -- we'll be installing the latest and greatest Kubernetes, so, that will change.

## So, what exactly do these playbooks do?

1. Configures a machine to use as a virtual machine host (which is optional, you'll get to choose this later on) on which the nodes run.
2. Installs all the deps necessary on the hosts
3. Runs `kubeadm init` to bootstrap the cluster ([kubeadm docs](https://kubernetes.io/docs/getting-started-guides/kubeadm/))
4. Installs a CNI plugin for pod networking (by default, it's flannel.)
5. Joins the hosts to a cluster.

## You chose the adventure: Provision a host that can run virtual machines

If you chose the adventure *"2. Spin up whatever CentOS hosts yourself."* head down to the next header topic, you've just saved yourself some work. (Unless you had to manually install CentOS like, twice, then you didn't but I'm hopeful you have a good way to spin up nodes in your environment.)

If you chose *"1. Provision a host that can run virtual machines."*, continue reading from here.

I recommended adventure #2, to spin them up yourself. I'm only going to glance over this part, I think it's handy for iterating on Kubernetes setups, but, there's really a bunch of options here. For the time being -- I'm going to only cover a setup that uses a NAT'd setup for the VMs. IMO -- it's less convenient, but, it's more normalized to generally document. So that's what we'll get today.

Alright -- so you've got CentOS all setup on this new host, and you can SSH to it, and at least sudo root from there. That's necessary for our Ansible playbook.

Let's create a small inventory, and we'll use that.

We can copy out a sample inventory, and we'll go from there.

```
$ cp inventory/examples/virthost/virthost.inventory inventory/your_virthost.inventory
```

All edited, mine looks like:

```
vmhost ansible_host=192.168.1.119 ansible_ssh_user=root

[virthost]
vmhost
```

This assumes you can SSH as root to that `ansible_host` specified there.

If you've got that all set -- it shouldn't be hard to spin up some VMs, now.

Just go ahead and run the `virthost-setup` playbook, such as:

```
$ ansible-playbook -i inventory/your_virthost.inventory -e "ssh_proxy_enabled=true" playbooks/virthost-setup.yml
```

By default this will spin up 4 hosts for us to use. If you'd like to use other hosts, you can specify them, you'll find the default variable for the list of these VMs in the variable called `virtual_machines` in the `./playbooks/ka-init/group_vars/all.yml` file, which you're intended to override (instead of edit) -- you can specify the memory & CPU requirements for those VMs, too.

Let that puppy run, and you'll find out that it will create a file for you with a new inventory -- `./inventory/vms.local.generated`. 

It has also created a private key to SSH to these vms. So if you want to ssh to one, you can do something like:

```
$ ssh -i ~/.ssh/vmhost/id_vm_rsa -o ProxyCommand="ssh -W %h:%p root@192.168.1.119" centos@192.168.122.58
```

Where:

* ` ~/.ssh/vmhost/id_vm_rsa` is the private key, and `vmhost` is the name of the host from the first inventory we used.
* `192.168.1.119` is the IP address of the virtualization host.
* and `192.168.122.58` is the IP address of the VM (which you discovered from looking at the `vms.local.generated` file)

Check that out, we're going to use it in the "Interall Kubernetes step" (which you can skip to, now.)

## You chose the adventure: Spin up whatever CentOS hosts yourself

If you chose *"1. Provision a host that can run virtual machines."*, continue to the next header.

Go ahead and spin up N+1 boxes. I recommend at least 2, 3 makes it more interesting. And even more for the brave. You need at least a master, and I recommend another as a node.

Make sure that you can SSH to these boxes, and let's create a sample inventory.

Create yourself an inventory, which you can base on this inventory:

```
kube-master ansible_host=192.168.122.216
kube-node-1 ansible_host=192.168.122.179
kube-node-2 ansible_host=192.168.122.32

[master]
kube-master

[nodes]
kube-node-1
kube-node-2

[all:vars]
ansible_user=centos
ansible_ssh_private_key_file=/home/me/.ssh/my_id_of_some_sort
```

Go ahead and put that inventory file in the `./inventory` directory at whatever name you choose, I'd choose `./inventory/myname.inventory` -- you can replace `myname` with your name, your dogs name, your favorite cheese -- actually that's the official suggested name of the inventory now... `manchego.inventory`.

So place that file at `./inventory/manchego.inventory`.

*(sidenote, I actually prefer a sharp cheddar, or a brie-style cheese like Jasper Hill's [Moses Sleeper](https://www.jasperhillfarm.com/moses/))*

## Installing Kubernetes

Alright -- you've gotten this far, you're on the path to success. Let's kick off an install.

Replace `./inventory/your.inventory` with:

* `./inventory/vms.local.generated` if you chose #1, build a virtualization host
* `./inventory/manchego.inventory` if you chose #2, provision your own machines.

```
$ ansible-playbook -i ./inventory/your.inventory playbooks/kube-install.yml
```

Wait! Did you already run that? If you didn't there's another mini-adventure you can choose, go to the next header, *"Run the `kube-install` with Multus for networking"*.

And you're on the way to success! And if you've finished your coffee now... It's time to skip down to *"Verify your Kubernetes setup!"*

## (Optional) Run the `kube-install` with Multus for networking

If you aren't going to use Multus, skip down to *"Verify your Kubernetes setup!"*, otherwise, continue here.

Alright, so this is an optional one, some of my audience for this blog gets here because they're looking for a way to use [Multus CNI](https://github.com/Intel-Corp/multus-cni). I'm a big fan of Multus, it allows us to attach multiple network interfaces to pods. If you're following Multus, I urge you to check out what's happening with the Network Plumbing Working Group (NPWG) -- an offshoot of Kubernetes SIG-Network (the special interest group for networking). Up in the NPWG, we're working on standardizing how multiple network attachments for pods work, and I'm excited to be trying Multus.

Ok, so you want to use Multus! Great. Let's create an extra vars file that we can use.

```
$ cat inventory/multus-extravars.yml 
---
pod_network_type: "multus"
multus_use_crd: false
optional_packages:
  - tcpdump
  - bind-utils
multus_ipam_subnet: "192.168.122.0/24"
multus_ipam_rangeStart: "192.168.122.200"
multus_ipam_rangeEnd: "192.168.122.216"
multus_ipam_gateway: "192.168.122.1"
```

Our Multus demo uses macvlan -- so you'll want to change the `multus_ipam_*` variables to match your network. This one matches the default NAT'ed setup for libvirt VMs in CentOS. 

Now that we have that file in place, we can kick off the install like so:

```
$ ansible-playbook -i ./inventory/vms.local.generated -e "@./inventory/multus-extravars.yml" playbooks/kube-install.yml
```

If you created your own inventory change `./inventory/vms.local.generated` with `./inventory/manchego.inventory` (or whatever you called yours if you didn't pick my cheesy inventory name).

## Verify your Kubernetes setup!

Go ahead and SSH to the master node, and you can view which nodes have registered, if everything is good, it should look something like:

```
[centos@kube-master ~]$ kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
kube-master   Ready     master    30m       v1.9.3
kube-node-1   Ready     <none>    22m       v1.9.3
kube-node-2   Ready     <none>    22m       v1.9.3
kube-node-3   Ready     <none>    22m       v1.9.3
```

Let's create a pod to make sure things are working a-ok.

Create a yaml file that looks like so:

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
```

And tell kube to create the pods with:

```
[centos@kube-master ~]$ kubectl create -f nginx_pod.yaml 
```

Watch them come up with:

```
[centos@kube-master ~]$ watch -n1 kubectl get pods -o wide
```

Assuming you have multiple nodes, these should be coming up on separate nodes, once they're up, go ahead and find the IP of one of them...

```
[centos@kube-master ~]$ IP=$(kubectl describe pod $(kubectl get pods | grep nginx | head -n1 | awk '{print $1}') | grep -P "^IP" | awk '{print $2}')
[centos@kube-master ~]$ echo $IP
10.244.3.2
[centos@kube-master ~]$ curl -s $IP | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

And there you have it, an instance of nginx running on Kube!

## For Multus verification...

(If you haven't installed with Multus, skip down to the *"Some other adventures you can choose"* section.)

You can kick off a pod and go ahead and exec `ip a` on it. The nginx pods that we spun up don't have the right tools to inspect the network. So let's kick off a pod with some better tools.

Create a yaml file like so:

```
[centos@kube-master ~]$ cat check_network.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: debugging
spec:
  containers:
    - name: debugging
      command: ["/bin/bash", "-c", "sleep 2000000000000"]
      image: dougbtv/centos-network
      ports:
      - containerPort: 80
```

Then have Kubernetes create that pod for you...

```
[centos@kube-master ~]$ kubectl create -f check_network.yaml 
```

You can watch it come up with `watch -n1 kubectl get pods -o wide`, then you can verify that it has multiple interfaces...

```
[centos@kube-master ~]$ kubectl exec -it debugging -- ip a | grep -Pi "^\d|^\s*inet\s"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    inet 127.0.0.1/8 scope host lo
3: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    inet 10.244.3.2/24 scope global eth0
4: net0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    inet 192.168.122.200/24 scope global net0
```

Hurray! There's your Kubernetes install up and running showing multiple network attachments per pod using Multus.

## Some other adventures you can choose...

This is just the tip of the iceberg for more advanced scenarios you can spin up...

* You can install multiple CNI (container network interface) plugins, such as Multus. [Check out the article on how to do so](http://dougbtv.com/nfvpe/2017/12/19/multus-crd/).
* You can have some persistent storage by following [this article on how to configure GlusterFS](http://dougbtv.com/nfvpe/2017/08/10/gluster-kubernetes/) with Kubernetes.

If you made the first decision in this article to install Kube, congrats! THE END.

## You have chosen: Do not install Kubernetes

It is pitch black. You are likely to be eaten by a grue. You have been eaten by a grue. THE END.