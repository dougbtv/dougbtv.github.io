---
author: dougbtv
comments: true
date: 2016-10-31 11:35:00-05:00
layout: post
slug: openshift-on-openstack-easy-mode
title: OpenShift-on-OpenStack - Part 1 - "Easy Mode"
---

A shift-on-a-stack-on-stack, it's [turtles all the way down](https://en.wikipedia.org/wiki/Turtles_all_the_way_down). It's cool! We're going to first use ["TripleO Quick Start" aka oooq](https://github.com/openstack/tripleo-quickstart) to get our openstack up. The gist here is that we'll use a single baremetal host that we'll call our "virtual machine host" which is going to have a VM to run the undercloud, and then in this simple example we'll use the defaults so that there's just one controller and one compute node.

I'm calling this "easy mode" because we're just going to use the "all-in-one" method of spinning up OpenShift to limit the scope initially, and we'll build on this in a following part. In following parts we'll also customize the networking configuration and look at HA.

You'll note my virtual machine host is named eendracht which is the first recorded vessel that [discovered Australia](https://en.wikipedia.org/wiki/Eendracht_(1615_ship)). So whenever you see that, just remember it means "your virtual machine host". I think I mostly cleaned that up here, but, in case!

One thing I ran into with my test machine, which is 3.1 ghz / 8 core AMD, 32 gigs of ram. I was using a big ole 2 TB spinning disk that is old, and slow. I really had it around for just handy temporary storage of large things. Well, I would run into an issue where during the undercloud install there would be these "db sync" operations which sync up MySQL and apparently dump a bunch of data in it. There'd be a timeout, and everything after that would fail in a nasty way. I went and chucked in a 480 gig solid state HDD, and that all went away. I went and talked to the folks in the `#oooq` channel on Freenode, and they were quite helpful at pointing me in the right place so that I could figure out that this was I/O bound. Maybe at some point I can help oooq look at that, too.

## Setup OpenStack using Triple-O Quick Start (oooq)

Configure some basics for the virtual machine host (on baremetal). According to your own preferences, the network setup, ssh keys to root, yum update (maybe install some utils you like on the virtual machine host [cough, net-tools, cough]). SSH keys like, y'know:

```
ssh-keygen -t rsa
ssh-copy-id root@machine
```

Client / control machine setup (e.g. your laptop)

```
git clone https://github.com/openstack/tripleo-quickstart.git
bash quickstart.sh --install-deps
```

Run quickstart

```
./quickstart.sh eendracht`
```

When it's finished let's snapshot that VM

```
virsh snapshot-create-as undercloud undercloud-installed-undercloud
```

Follow [steps for overcloud deploy](http://ow.ly/c44w304begR).

```
ssh -F /home/doug/.quickstart/ssh.config.ansible undercloud
source stackrc
openstack overcloud image upload
openstack baremetal import instackenv.json
openstack baremetal configure boot
openstack baremetal introspection bulk start
```

A little post setup. We'll setup subnet DNS

```
subnet_id=$(neutron subnet-list | grep -i "start" | awk '{print $2}')
neutron subnet-update $subnet_id --dns-nameserver 8.8.8.8
```

Drum roll please, here's where the magic happens! Not really magic, but, where we'll get an overcloud to deploy compute instances on.

```
openstack overcloud deploy --templates
```

Snapshot all the things (so just in case something goes wrong on one or more of these VMs we can revert)

* `virsh snapshot-create-as undercloud undercloud-installed-overcloud`
* Should you need to revert to a snapshot.
  * `virsh snapshot-revert --domain undercloud undercloud-installed-undercloud`

* Connecting to your box
  * You can SSH to the undercloud
    * `ssh -F ~/.quickstart/ssh.config.ansible stack@undercloud`
  * And if you wanna tunnel in and use the dashboard GUI
    * `ssh -F ~/.quickstart/ssh.config.ansible stack@undercloud -L 8080:192.0.2.6:80`
    * Then you can visit [http://localhost:8080/](http://localhost:8080/)

## Neutron network setup

I was getting this backwards for quite some time, but, luckily I got it to work with a lot of [great help from this blog post with a complete run-through of oooq](http://dbaxps.blogspot.com/2016/10/tripleo-quickstart-deployment-rdo.html).

Here's our network & subnet creation

```
neutron net-create ext-net --router:external --provider:physical_network datacentre  --provider:network_type flat
neutron subnet-create ext-net --name ext-subnet --allocation-pool start=192.0.2.100,end=192.0.2.120 --disable-dhcp --gateway 192.0.2.1  192.0.2.0/24
neutron router-create router1
neutron router-gateway-set router1 ext-net
neutron net-create int
neutron subnet-create int 30.0.0.0/24 --dns_nameservers list=true 8.8.8.8
neutron router-interface-add router1 $ABOVE_ID
```

Now, create security group rules, we'll do SSH & ICMP for now.

```
nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```

## Boot some instances for OpenShift to run on

We'll make two, but, we're just going to use one for the example now, in "easy mode"

So we want to [install OpenShift](https://install.openshift.com/), huh? For now, we're going to go with the [latest development release](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md) install method. Downstream uses the [atomic-openshift-installer](https://docs.openshift.com/container-platform/latest/install_config/install/quick_install.html) which is, of course, RHEL based.

Let's get an [atomic host image](https://wiki.centos.org/SpecialInterestGroup/Atomic/Download/) up and running on it. Download image and upload to glance:

```
wget http://cloud.centos.org/centos/7/atomic/images/CentOS-Atomic-Host-7-GenericCloud.qcow2.gz
gunzip CentOS-Atomic-Host-7-GenericCloud.qcow2.gz
source overcloudrc
mv CentOS-Atomic-Host-7-GenericCloud.qcow2 /tmp/
glance image-create --name centos-atomic --disk-format qcow2  --container-format bare --visibility public --file /tmp/CentOS-Atomic-Host-7-GenericCloud.qcow2 --progress
```

We're gonna need a key pair, here's [how to do it](http://docs.openstack.org/user-guide/cli-nova-configure-access-security-for-instances.html)

```
nova keypair-add atomic_key > atomic_key.pem
chmod 0600 atomic_key.pem
```

Spin up a couple instances

```
internal_net=$(neutron net-list | awk ' /internal/ {print $2;}')
nova boot --flavor m1.small --key-name atomic_key --nic net-id=$internal_net --image centos-atomic atomic1
nova boot --flavor m1.small --key-name atomic_key --nic net-id=$internal_net --image centos-atomic atomic2
nova list
```

Associate floating IPs via GUI if you want, or at the CLI...

```
neutron floatingip-create ext-net
nova add-floating-ip atomic1 $NEW_IP
```

SSH to an instance, to make sure it works over the network like this. And as an example of how to use your keys. You'll notice you're using the `centos` user, which is the default username in a CentOS atomic host.

```
ssh -i atomic_key.pem centos@192.0.2.101
```

## Now let's install OpenShift

First you're going to [download the OpenShift Client (oc) from GitHub](https://github.com/openshift/origin/releases). Remember, you need client version >= 1.3

```
ssh -i atomic_key.pem centos@192.0.2.101
curl -o /tmp/oc.tar.gz -L https://github.com/openshift/origin/releases/download/v1.3.1/openshift-origin-client-tools-v1.3.1-dad658de7465ba8a234a4fb40b5b446a45a4cee1-linux-64bit.tar.gz
mkdir -p /tmp/oc-tools` 
tar -xzvf /tmp/oc.tar.gz -C /tmp/oc-tools
mkdir ~/bin/
cp -a /tmp/oc-tools/openshift-origin-client-tools-v1.3.1-dad658de7465ba8a234a4fb40b5b446a45a4cee1-linux-64bit/oc ~/bin/
```

Wait, what's the deal with the `~/bin` directory? That's already in your path, in an atomic host, btw. This is likely because your root file system is read-only, so this gives you a convenient place to put customizable things, such as... The OpenShift client.

Then, allow centos user to use docker

```
sudo groupadd docker
sudo gpasswd -a centos docker
sudo systemctl restart docker
newgrp docker
docker ps
```

Configure an insecure registry

```
sudo bash -c "echo \"INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'\" >> /etc/sysconfig/docker"
sudo systemctl restart docker
```

Now run oc cluster up -- if you've gotten here, it's looking really good. This is the true install of OpenShift, now. It's going to pull the Docker images you need and get 'em all running for you.

```
oc cluster up
```

Let's let that machine talk, add port 8443 and some spares to default security group
```
nova secgroup-add-rule default tcp 8443 8443 0.0.0.0/0
nova secgroup-add-rule default tcp 80 80 0.0.0.0/0
nova secgroup-add-rule default tcp 443 443 0.0.0.0/0
```

And of course you can use the `oc` (openshift command) to manage your cluster. So you might want to check out how to [get started with the OpenShift CLI](https://docs.openshift.com/enterprise/3.0/cli_reference/get_started_cli.html).


What about next time? We'll make it a true OpenShift cluster, because this isn't very much fun with just a single stand-alone all-in-one node. That's not what we're trying to accomplish here, we want all the good stuff. So we'll look at potentially using [openshift-ansible](https://github.com/openshift/openshift-ansible) and we might even reference this blog's example of having [openshift-ansible in action on OpenStack](https://blog.zhaw.ch/icclab/installing-openshift-origin-v3-on-openstack/).
