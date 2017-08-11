---
author: dougbtv
comments: true
date: 2017-08-10 16:00:00-05:00
layout: post
slug: gluster-kubernetes
title: Be a hyper spaz about a hyperconverged GlusterFS setup with dynamically provisioned Kubernetes persistent volumes
category: nfvpe
---

I'd recently brought up my [GlusterFS for persistent volumes in Kubernetes setup](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/) and I was noticing something errant. I had to REALLY baby the persistent volumes. That didn't sit right with me, so I refactored the setup to use [gluster-kubernetes](https://github.com/gluster/gluster-kubernetes) to hook up a hyperconverged setup. This setup improves on the previous setup by both having the Gluster daemon running in Kubernetes pods, which is just feeling [so fresh and so clean](https://www.youtube.com/watch?v=-JfEJq56IwI). Difference being that OutKast is like smooth and cool -- and I'm [an excited spaz](http://www.dailymotion.com/video/xhl09y) about technology with this. Gluster-Kubernetes also implements [heketi](https://github.com/heketi/heketi) which is an API for GlusterFS volume management -- that Kube can also use to allow us dynamic provisioning. Our goal today is to spin up Kube (using [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible)) with gluster-kubernetes for dynamic provisioning, and then we'll validate it with master-slave replication in MySQL, to one-up our simple MySQL from the last article.

If you're not familiar with persistent volumes in Kubernetes, or some of the basics of why GlusterFS is pretty darn cool -- give my [previous article](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/) a read for those basics. But, come back here for the setup.

The bulk of the work I was able to do here was thanks to the [gluster-kubernetes setup guide](https://github.com/gluster/gluster-kubernetes/blob/master/docs/setup-guide.md), which helps you use the tool embedded in that project called [gk-deploy](https://github.com/gluster/gluster-kubernetes/blob/master/deploy/gk-deploy). This article (and the playbook) leans on gk-deploy quite a bit. I'd also like to thank [@jarrpa](https://github.com/jarrpa) for some help he gave when I ran into some documentation snags bringing up gluster-kubernetes.

## Requirements

In short, I recommend my usual setup which is a single CentOS 7 machine you can run VMs on. That's what I typically use with kube-centos-ansible. You're going to need approximately 100 gigs of disk. You'll run 4 virtual machines (one master and 4 minions). I personally use 2 vCPUs per VM, you'd likely get away with one. 

Otherwise, you can also use this on baremetal, just skip the VM portion of kube-centos-ansible. The tricky part is the that kube-centos-ansible currently only supports a single disk for this setup, and it'd need to be the same name on all baremetal hosts. If you do give it a go, just change the name of the disk in the `spare_disk_dev` in the `./group_vars/all.yml` in your kube-centos-ansible clone. And, you'll need some disks that are free-and-clear of data, and not mounted on your machines. kube-centos-ansible can set this up for you in VMs. I'm also happy to take some pull requests to improve how this works against baremetal!

Also, as per usual, I assume a CentOS 7 distro on all nodes. And while you might be able to do this with other distros that it colors how I approach this and what ancillary tools I select.

Lastly, you need a client machine you can run Ansible on, and must have Ansible installed. 

## But, why? Isn't the previous article's method just fine?

First and foremost -- the [original article](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/) I wrote didn't have [heketi](https://github.com/heketi/heketi) -- the API that we're going to have Kube use to dynamically provision Gluster volumes. That's not as good.

The other thing was cleanliness. It was kind of two ways of managing applications -- one running on the host operating system, and the others in containers. Just not nearly as clean.

Lastly, it required that you baby some of the volumes. For example, you'd have to specify new persistent volumes, and then make claims against them. Now we can have claims against a new Kubernetes [storageclass](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#storageclasses), and that storage class will specify that we talk to Heketi, [like in this example](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#glusterfs).

Also, we use the `gk-deploy` tool from gluster-kubernetes here, and it can do a number of things that we just don't have to maintain anymore -- such as "peer probe" all the gluster nodes; which gets them all connected to one another and cooperating.

This begs the question -- is there an advantage to running it on the host? I don't think there is. This has all the pieces that has, it just happens to have them running in containers on the host. Since you're running Kubernetes -- I think that's an advantage. 

It should be noted however that the `gk-deploy` tool also supports using an existing GlusterFS cluster, and it can just run heketi for us. (However, my playbook doesn't intend to support that mode, for now.)

## Kubernetes Installation (the hard part)

I'll give a quick review of [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible). If you want [a more thorough tutorial](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) check out my article on using it. The most difficult part is just modifying the inventory, and that's not even that tough. Remember the gist here is that we have a single host that can run virtual machines (which we call the "virthost", and this playbook has the setup for running those), and then we run virtual machines on which we run Kubernetes (generally for laboratory analysis, in my own case).

Clone up the kube-centos-ansible repo (at a particular tag that has the kube-glusterfs):

```
$ git clone --branch v0.1.0 https://github.com/dougbtv/kube-centos-ansible.git && cd kube-centos-ansible
```

Now go and do the hardest part. Modify the inventories. Modify `./inventory/virthost.inventory` to your main CentOS machine to run virtual machines on. Add a vars section to the bottom of it:

```
[kubehost:vars]
bridge_physical_nic=eth0
bridge_network_cidr=192.168.1.0/24
```

And set the `eth0` to whatever your primary NIC is named (e.g. if you have multiple NICs, it's likely in your lab this would be the NIC that can access the internet). And set the CIDR for it too. Of course, at the top set the IP address of this host.

Now we'll run the virthost setup:

```
$ ansible-playbook -i inventory/virthost.inventory virt-host-setup.yml
```

Two things you need to do from here:

* Pay attention to the list of IPs for the VMs that come up in a play described as: `Here are the IPs of the VMs`
* Next, go ahead and get the contents of `/root/.ssh/id_vm_rsa` (the SSH private key) on the virt host. Put those somewhere so on your client machine (workstation or what have you)

Modify the `./inventory/vms.inventory`. In the first four lines, put the IP addresses you got from the last step. Then, the last line point the `ansible_ssh_private_key_file` variable at the path to the SSH private key you got from the previous step. And lastly -- comment out the `ansible_ssh_common_args` line, you don't need that now.

Now you can install Kubernetes.

```
$ ansible-playbook -i inventory/vms.inventory kube-install.yml
```

To verify it, on the virt host you can ssh to the kube master, like so, and get the list of nodes:

```
$ ssh -i .ssh/id_vm_rsa centos@kube-master 'kubectl get nodes'
```

Cool -- now you have Kube. We're going to attach some spare disks to those VMs which will show up as `/dev/vdb` on each of them. By default they're 10 gigs (and you can change that in the `spare_disk_size_megs` variable in `./group_vars/all.yml` or put it in your inventory)

```
ansible-playbook -i inventory/virthost.inventory vm-attach-disk.yml
```

Alright, you're good to go -- now onto the good stuff.

## GlusterFS on Kube (the easy part)

Here's the easy part -- just one more playbook to run. Then we can go from there.

```
$ ansible-playbook -i inventory/vms.inventory gluster-install.yml
```

This is going to do everything you need to have glusterfs running on each of the minion nodes.

The (at least mock) hyperconverged storage situation is coming now. If you're not familiar with that terminology -- the shortest explanation is that your storage resides on the same hosts as where you run your computational workloads. Awesome.

Great -- that's a whole bunch of magic, what the heck did that playbook actually do!? If you want to see it in stark detail, checkout the `./roles/glusterfs-kube-config/tasks/main.yml` file which has all of what it does.

Here's the run-down:

* Installs some required packages (`glusterfs-fuse` is required on all nodes)
* Templates a `gk-deploy` topology file, from `./roles/glusterfs-kube-config/templates/glusterfs-topology.json.j2` 
    - You can also check out an example, if you'd like.
* Clones [gluster-kubernetes](https://github.com/gluster/gluster-kubernetes)
* Installs the [heketi CLI application](https://github.com/heketi/heketi/releases) on the kube master.
* Runs the `gk-deploy` script
    - Using the topology file we templated
    - Specifying that we'll run GlusterFS daemon in Kubernetes
* Creates a storageclass from a template in `./roles/glusterfs-kube-config/templates/glusterfs-storageclass.yaml.j2`

It's actually a LOT less steps than before. Primarily because we don't have to worry about such things as:

* Formatting disks and creating volume groups, etc.
* Configuring GlusterFS more deeply and manually peering the endpoints.
* ...and more.

## Let's use it!

Alright cool, well, you just hung out for a while waiting for that GlusterFS playbook to run (not to mention, an entire Kubernetes install). Which makes me believe that you're sufficiently coffee-i-fied at this point. Because of that, we're going to pick something a little bit more ambitious this time for an example usage of these persistent volumes. Last time we used MariaDB, this time, we're going to use MySQL with replication.

## Setting up MySQL replication in Kubernetes

If you're interested more deeply in how to do this, check out the [k8s docs on running replicated mysql using stateful sets](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/). That's the origin of my example, but, I have some modified resource definitions here that are specific to what we just spun up so you don't have to read through every line. However, it is actually fairly interesting to check out, so I do encourage it.

Firstly, let's curl down those resource definitions. I also [have them in a GitHub Gist](https://gist.github.com/dougbtv/19dca51388d74a915bf690bdb2e12dfd).

Ok, let's get the files.

```
$ curl -s -L https://goo.gl/SKqo8J > mysql-configmap.yaml
$ curl -s -L https://goo.gl/msDQTb > mysql-services.yaml
$ curl -s -L https://goo.gl/doH2gN > mysql-statefulset.yaml
```

Create from all of those.

```
$ kubectl create -f mysql-configmap.yaml
$ kubectl create -f mysql-services.yaml
$ kubectl create -f mysql-statefulset.yaml
```

(One time I had to recreate the stateful set, MySQL complained that I couldn't connect from an arbitrary IP address one time. Unsure what caused that, but if it happens to you just `kubectl delete -f mysql*.yml` and try again.
)

It takes a bit to spin up, since it's a stateful set, the pods come up ordered for us, which is nice for a replicated setup. So make sure to do a `watch -n1 kubectl get pods` (or even a `kubectl get pods --watch`).

## Verifying the MySQL setup.

Now, we can do cool stuff with it. Let's create a table based on... Honey bees (I keep bees but these numbers aren't representative of anything scientific, just FYI). Feel free to use whatever data you'd like.

```
[centos@kube-master ~]$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- mysql -h mysql-0.mysql
mysql> CREATE DATABASE beekeeping;
mysql> USE beekeeping;
mysql> CREATE TABLE hive (id INT AUTO_INCREMENT, role VARCHAR(255), counted BIGINT, PRIMARY KEY (id));
mysql> INSERT INTO hive VALUES (NULL,'queen',1);
mysql> INSERT INTO hive VALUES (NULL,'worker',20000);
mysql> INSERT INTO hive VALUES (NULL,'drone',800);
mysql> SELECT * FROM hive;
```

Ok, that's all well and good, now, let's check that the replicated members have data.

```
[centos@kube-master ~]$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- mysql -h mysql-read --execute "SELECT * FROM beekeeping.hive"
+----+--------+---------+
| id | role   | counted |
+----+--------+---------+
|  1 | queen  |       1 |
|  2 | worker |   20000 |
|  3 | drone  |     800 |
+----+--------+---------+
```

Now, let's have fun and tear it down, and see if we still have data rollin'.

```
[centos@kube-master ~]$ kubectl delete -f mysql-statefulset.yaml 
[centos@kube-master ~]$ kubectl create -f mysql-statefulset.yaml 
```

And then exec the select again, and bammo...

```
[centos@kube-master ~]$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never -- mysql -h mysql-read --execute "SELECT * FROM beekeeping.hive"
+----+--------+---------+
| id | role   | counted |
+----+--------+---------+
|  1 | queen  |       1 |
|  2 | worker |   20000 |
|  3 | drone  |     800 |
+----+--------+---------+
```

You're cookin' with oil!

