---
author: dougbtv
comments: true
date: 2017-04-05 12:05:08-05:00
layout: post
slug: glusterfs-persistent
title: How-to use GlusterFS to back persistent volumes in Kubernetes
category: nfvpe
---

A mountain I keep walking around instead of climbing in my Kubernetes lab is storing persistent data, I kept avoiding it. Sure -- in a lab, I can just throw it all out most of the time. But, what about when we really need it? I decided I would use [GlusterFS](https://www.gluster.org/) to back my [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and I've got to say... My experience with GlusterFS was great, I really enjoyed using it, and it seems rather resilient -- and best of all? It was pretty easy to get going and to operate. Today we'll spin up a Kubernetes cluster using my [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks, and use some newly included plays that also setup a GlusterFS cluster. With that in hand, our goal will be to setup the persistent volumes and claims to those volumes, and we'll spin up a MariaDB pod that stores data in a persistent volume, important data that we want to keep -- so we'll make some data about Vermont beer as it's very very important.

## Requirements

First up, this article will use my [spin-up Kubernetes on CentOS article](http://dougbtv.com//nfvpe/2017/02/16/kubernetes-1.5-centos/) as a basis. So if there's any details you feel are missing from here -- make sure to double check that article as it goes further in depth for the moving parts that make up the [kube-centos-ansible](https://github.com/dougbtv/kube-centos-ansible) playbooks. Particularly there's more detail that article on how to modify the inventories and what's going on there, too (and where your ssh keys are, which you'll need too).

Now, what you'll need...

* A CentOS 7.3 host capable of spinning up a few virtual machines
* A machine where you can run Ansible (which can also be the same host if you like), and has git to clone our playbooks.

That's what we're going to base it on. If you'd rather not use virtual machines, that's OK! But, if you choose to spin this up on bare metal, you'll have to do all the OS install yourself (as you guessed, or maybe you're all cool and using Bifrost or Spacewalk or something cool, and that's great too). To make it interesting, I'd recommend at least 3 hosts (a master and two minions), and... There's one more important part you're going to have to do if you're using baremetal -- and that's going to be to make sure there's a couple empty partitions available. Read ahead first and see what it looks like here with the VMs. Those partitions you'll have to format for GlusterFS. In fact -- that's THE only hard part of this whole process is that you've gotta have some empty partitions across a few hosts that you can modify.

## Let's get our Kubernetes cluster running.

Ok, step zero -- you need a clone of my playbooks, so make a clone and move into it's directory...

    git clone https://github.com/dougbtv/kube-centos-ansible.git

Since we've got that we're going to do run the `virt-host-setup.yml` playbook which sets up our CentOS host so that it can create a few virtual machines. The defaults spin up 4 machines, and you can modify some of these preferences by going into the `vars/all.yml` if you please. Also, you'll need to modify the `inventory/virthost.inventory` file to suit your environment.

    ansible-playbook -i inventory/virthost.inventory virt-host-setup.yml

Once that is complete, on your virtual machine host you should see some machines running if you were to run

    virsh list --all

The `virt-host-setup` will complete with a set of IP addresses, so go ahead and use those in the `inventory/vms.inventory` file, and then we can start our Kubernetes installation.

    ansible-playbook -i inventory/vms.inventory kube-install.yml

You can check that Kubernetes is running successfully now, SSH into your Master (you'll need to do other work there soon, too)

    kubectl get nodes

And you should see the master and 3 minions, by default. Alright, Kubernetes is up.

## Let's get GlusterFS running!

So we're going to use a few playbooks here to get this all setup for you. Before we do that, let me speak to what's happening in the background, and we'll take a little peek for ourselves with our setup up and running.

First of all, most of my work to automate this with Ansible was based on this [article on installing a GlusterFS cluster on CentOS](https://wiki.centos.org/HowTos/GlusterFSonCentOS), which I think comes from Storage SIG (maybe). I also referenced this [blog article from Gluster about GlusterFS with Kubernetes](http://blog.gluster.org/2016/03/persistent-volume-and-claim-in-openshift-and-kubernetes-using-glusterfs-volume-plugin/). Last but not least, there's [example implementations of GlusterFS From Kubernetes GitHub repo](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/glusterfs).

Next a little consideration for your own architectural needs is that we're going to use the Kubernetes nodes as GlusterFS themselves. Additionally -- then we're running GlusterFS processes on the hosts themselves. So, this wouldn't work for an all Atomic Host setup. Which is unfortunate, and I admit I'm not entirely happy with it, but it might be a [premature optimization](http://wiki.c2.com/?PrematureOptimization) of sorts right now. However, this is more-or-less a proof-of-concept. If you're so inclined, you might be able to modify what's here to adapt it to a fully containerized deployment (it'd be a lot, A LOT, swankier). You might want to organize this otherwise, but, it was convenient to make it this way, and could be easily broken out with a different inventory scheme if you so wished.

### Attach some disks

The first playbook we're going to run is the `vm-attach-disk` playbook. This is based on [this publicly available help from access.redhat.com](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-Virtualization-Virtualized_block_devices-Adding_storage_devices_to_guests.html). The gist is that we create some qcow images and attach them to our running guests on the virtual machine host. 

Let's first look at the devices available on our `kube-master` for instance, so list the block devices...

    [centos@kube-master ~]$ lsblk | grep -v docker

You'll note that there's just a vda mounted on `/`.

Let's run that playbook now and take a peek at it again after.

    ansible-playbook -i inventory/virthost.inventory vm-attach-disk.yml

Now go ahead and look on the master again, and list those block devices.

    [centos@kube-master ~]$ lsblk | grep -v docker

You should have a `vdb` that's 4 gigs. Great, that's what the playbook does, it does it across the 4 guests. You'll note it's not mounted, and it's not formatted.

### Configure those disks, and install & configure Gluster!

Now that our disks are attached, we can go ahead and configure Gluster.

    ansible-playbook -i inventory/vms.inventory gluster-install.yml

Here's what's been done:

* Physical volumes and volume groups created on those disks.
* Disks formatted as XFS.
* Partitions mounted on `/bricks/brick1` and `/bricks/brick2`.
* Nodes attached to GlusterFS cluster.
* Gluster volumes created across the cluster.
* Some yaml for k8s added @ `/home/centos/glusterfs*yaml`

Cool, now that it's setup let's look at a few things. We'll head right to the master to check this out.

```
[root@kube-master centos]# gluster peer status
```

You should see three peers with a connected state. Additionally, this should be the case on all the minions, too. 

```
[root@kube-minion-2 centos]# gluster peer status
Number of Peers: 3

Hostname: 192.168.122.14
Uuid: 3edd6f2f-0055-4d97-ac81-4861e15f6e49
State: Peer in Cluster (Connected)

Hostname: 192.168.122.189
Uuid: 48e2d30b-8144-4ae8-9e00-90de4462e2bc
State: Peer in Cluster (Connected)

Hostname: 192.168.122.160
Uuid: d20b39ba-8543-427d-8228-eacabd293b68
State: Peer in Cluster (Connected)
```

Lookin' good! How about volumes available?

```
[root@kube-master centos]# gluster volume status
```

Which will give you a lot of info about the volumes that have been created.

### Let's try using GlusterFS (optional)

So this part is entirely optional. But, do you want to see the filesystem in action? Let's temporarily mount a volume, and we'll write some data to it, and see it appear on other hosts.

```
[root@kube-master centos]# mkdir /mnt/gluster
[root@kube-master centos]# ipaddr=$(ifconfig | grep 192 | awk '{print $2}')
[root@kube-master centos]# mount -t glusterfs $ipaddr:/glustervol1 /mnt/gluster/
```

Ok, so now we have a gluster volume mounted at `/mnt/gluster` -- let's go ahead and put a file in there.

```
[root@kube-master centos]# echo "foo" >> /mnt/gluster/bar.txt
```

Now we should have a file, `bar.txt` with the contents "foo" on all the nodes in the `/bricks/brick1/brick1` directory. Let's verify that on a couple nodes.

```
[root@kube-master centos]# cat /bricks/brick1/brick1/bar.txt 
foo
```

And on kube-minion-2...

```
[root@kube-minion-2 centos]# cat /bricks/brick1/brick1/bar.txt
foo
```

Cool! Nifty right? Now let's clean up.

```
[root@kube-master centos]# rm /mnt/gluster/bar.txt
[root@kube-master centos]# umount /mnt/gluster/
```

## Add the persistent volumes to Kubernetes!

Alright, so you're all good now, it works! (Or, I hope it works for you at this point, from following 1,001 blog articles for how-to documents, I know sometimes it can get frustrating, but... I'm hopeful for you).

With that all set and verified a bit, we can go ahead and configure Kubernetes to get it all looking good for us.

On the master, as the centos user, look in the `/home/centos/` dir for these files...

```
[centos@kube-master ~]$ ls glusterfs-* -lah
-rw-rw-r--. 1 centos centos  781 Apr 19 19:08 glusterfs-endpoints.json
-rw-rw-r--. 1 centos centos  154 Apr 19 19:08 glusterfs-service.json
-rw-rw-r--. 1 centos centos 1.6K Apr 19 19:11 glusterfs-volumes.yaml
```

Go ahead and inspect them if you'd like. Let's go ahead and implement them for us.

```
[centos@kube-master ~]$ kubectl create -f glusterfs-endpoints.json 
endpoints "glusterfs-cluster" created
[centos@kube-master ~]$ kubectl create -f glusterfs-service.json 
service "glusterfs-cluster" created
[centos@kube-master ~]$ kubectl create -f glusterfs-volumes.yaml 
persistentvolume "gluster-volume-1" created
persistentvolume "gluster-volume-2" created
persistentvolume "gluster-volume-3" created
persistentvolume "gluster-volume-4" created
persistentvolume "gluster-volume-5" created
```

Now we can ask `kubectl` to show us the persistent volumes `pv`.

```
[centos@kube-master ~]$ kubectl get pv
NAME               CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
gluster-volume-1   600Mi      RWO           Delete          Available             storage                  18s
gluster-volume-2   300Mi      RWO           Delete          Available             storage                  18s
gluster-volume-3   300Mi      RWO           Delete          Available             storage                  18s
gluster-volume-4   100Mi      RWO           Delete          Available             storage                  18s
gluster-volume-5   100Mi      RWO           Delete          Available             storage                  18s
```

Alright! That's good now, we can go ahead and put these to use.

## Let's create our claims

First, we're going to need a persistent volume claim. So let's craft one here, and we'll get that going. The persistent volume claim is like "[staking a claim](https://en.wikipedia.org/wiki/Land_claim)" of land. We're going to say "Hey Kubernetes, we need a volume, and it's going to be this big". And it'll allocate it smartly. And it'll let you know for sure if there isn't anything that it can claim.

So create a file like so...

```
[centos@kube-master ~]$ cat pvc.yaml 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  creationTimestamp: null
  name: mariadb-data
spec:
  storageClassName: storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
status: {}
```

And then use `kubectl create` to apply it.

```
[centos@kube-master ~]$ kubectl create -f pvc.yaml 
persistentvolumeclaim "mariadb-data" created
```

And now we can list the "persistent volume claims", the `pvc`

```
[centos@kube-master ~]$ kubectl get pvc
NAME           STATUS    VOLUME             CAPACITY   ACCESSMODES   STORAGECLASS   AGE
mariadb-data   Bound     gluster-volume-2   300Mi      RWO           storage        20s
```

You'll see that Kubernetes was smart about it, and of the volumes we created -- it used juuuust the right one. We had a 600 meg claim, 300 meg claims, and a couple 100 meg claims. It picked the 300 meg claim properly. Awesome!

## Now, let's put those volumes to use in a Maria DB pod.

Great, now we have some storage we can use across the cluster. Let's go ahead and use it. We're going to use Maria DB cause it's a great example of a real-world way that we'd want to persist data -- in a database.

So let's create a YAML spec for this pod. Make yours like so:

```
[centos@kube-master ~]$ cat mariadb.yaml 
---
apiVersion: v1
kind: Pod
metadata:
  name: mariadb
spec:
  containers:
  - env:
      - name: MYSQL_ROOT_PASSWORD
        value: secret
    image: mariadb:10
    name: mariadb
    resources: {}
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: mariadb-data
  restartPolicy: Always
  volumes:
  - name: mariadb-data
    persistentVolumeClaim:
      claimName: mariadb-data
```

Cool, now create it...

    [centos@kube-master ~]$ kubectl create -f mariadb.yaml 

Then, watch it come up...

    [centos@kube-master ~]$ watch -n1 kubectl describe pod mariadb 

## Let's make some persistent data! Err, put beer in the fridge.

Once it comes up, let's go ahead and create some data in there we can pull back up. (If you didn't see it in the pod spec, you'll want to know that the password is "secret" without quotes).

This data needs to be important right? Otherwise, we'd just throw it out. So we're going to create some data regarding beer.

You'll note I'm creating a database called `kitchen` with a table called `fridge` and then I'm inserting some of the BEST beers in Vermont (and likely the world, I'm not biased! ;) ). Like Heady Topper from [The Alchemist](https://alchemistbeer.com/), and Lawson's [sip of sunshine](https://www.lawsonsfinest.com/beers/sip-sunshine/), and the best beer ever created -- [Hill Farmstead's Edward](http://hillfarmstead.com/ancestral-series/)

```
[centos@kube-master ~]$ kubectl exec -it mariadb -- /bin/bash
root@mariadb:/# stty cols 150
root@mariadb:/# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 10.1.22-MariaDB-1~jessie mariadb.org binary distribution

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE kitchen;
Query OK, 1 row affected (0.02 sec)

MariaDB [(none)]> USE kitchen;
Database changed
MariaDB [kitchen]> 
MariaDB [kitchen]> 
MariaDB [kitchen]> CREATE TABLE fridge (id INT AUTO_INCREMENT, item VARCHAR(255), quantity INT, PRIMARY KEY (id));
Query OK, 0 rows affected (0.31 sec)

MariaDB [kitchen]> INSERT INTO fridge VALUES (NULL,'heady topper',6);
Query OK, 1 row affected (0.05 sec)

MariaDB [kitchen]> INSERT INTO fridge VALUES (NULL,'sip of sunshine',6);
Query OK, 1 row affected (0.04 sec)

MariaDB [kitchen]> INSERT INTO fridge VALUES (NULL,'hill farmstead edward',6); 
Query OK, 1 row affected (0.03 sec)

MariaDB [kitchen]> SELECT * FROM fridge;
+----+-----------------------+----------+
| id | item                  | quantity |
+----+-----------------------+----------+
|  1 | heady topper          |        6 |
|  2 | sip of sunshine       |        6 |
|  3 | hill farmstead edward |        6 |
+----+-----------------------+----------+
3 rows in set (0.00 sec)
```

## Destroy the pod!

Cool -- well that's all well and good, we know there's some beer in our `kitchen.fridge` table in MariaDB.

But, let's destroy the pod, first -- where is the pod running, which minion? Let's check that out. We're going to restart it until it appears on a different node. (We could create an anti-affinity and all that good stuff, but, we'll just kinda jimmy it here for a quick demo.)

```
[centos@kube-master ~]$ kubectl describe pod mariadb | grep -P "^Node:"
Node:       kube-minion-2/192.168.122.43
```

Alright, you'll see mine is running on `kube-minion-2`, let's remove that pod and create it again.

```
[centos@kube-master ~]$ kubectl delete pod mariadb
pod "mariadb" deleted
[centos@kube-master ~]$ kubectl create -f mariadb.yaml 
[centos@kube-master ~]$ watch -n1 kubectl describe pod mariadb 
```

Watch it come up again, and if it comes up on the same node -- delete it and create it again. I believe it happens round-robin-ish, so... It'll probably come up somewhere else.

Now, once it's up -- let's go and check out the data in it.

```
[centos@kube-master ~]$ kubectl exec -it mariadb -- mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 2
Server version: 10.1.22-MariaDB-1~jessie mariadb.org binary distribution

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> USE kitchen;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [kitchen]> SELECT * FROM fridge;
+----+-----------------------+----------+
| id | item                  | quantity |
+----+-----------------------+----------+
|  1 | heady topper          |        6 |
|  2 | sip of sunshine       |        6 |
|  3 | hill farmstead edward |        6 |
+----+-----------------------+----------+
3 rows in set (0.00 sec)
```

Hurray! There's all the beer still in the fridge. Phew!!! Precious, precious beer.