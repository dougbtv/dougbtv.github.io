---
author: dougbtv
comments: true
date: 2016-11-03 13:42:00-05:00
layout: post
slug: custom-centos-cloud-image
title: Creating a Custom Centos Cloud Image (for OpenStack)
---

So I'm working on getting [openshift-ansible](https://github.com/openshift/openshift-ansible) to spin up on top of OpenStack. And I've run into a few snags, one of which requires that I need to make a custom cloud image to run because the [CentOS Generic Cloud image](http://cloud.centos.org/centos/7/images/) doesn't have NetworkManager install/enabled, and... Even worse but maybe a story for another day is that it doesn't support a [Centos Atomic Host](https://wiki.centos.org/SpecialInterestGroup/Atomic/Download/). Another approach would be to try to improve the Ansible script in openshift-ansible, however, this slices the [Gordian Knot](https://en.wikipedia.org/wiki/Gordian_Knot) for now, and we'll come back to that another time (as there's plenty else to let ansible-openshift know about as you'll find out in an upcoming article).

Looking on the bright side -- bonus! We can do other stuff to the image to customize it ourselves, too. I didn't do a whole lot, but, it feels nice at any rate.

We'll be following this [guide on making a custom centos cloud image](http://docs.openstack.org/image-guide/centos-image.html) from the official OpenStack docs. 

I'm going to do all of this from my laptop (which runs Fedora 25 [I always wanna write 'FC25' but, that's ancient history]), and then upload the image to my undercloud, and then to glance, and then run an instance to test it out. 

You might need a few tools...

```
[doug@localhost tmp]$ sudo dnf install -y @virtualization
[doug@localhost tmp]$ sudo dnf install -y virt-install libvirt-client
```

Download the minimal ISO

```
[doug@localhost Downloads]$ wget http://mirror.math.princeton.edu/pub/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso
[doug@localhost Downloads]$ mv CentOS-7-x86_64-Minimal-1511.iso /tmp/
[doug@localhost Downloads]$ cd /tmp/
[doug@localhost tmp]$ qemu-img create -f qcow2 /tmp/centos.qcow2 10G
Formatting '/tmp/centos.qcow2', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16

```

Now run a virtual machine... Before I did this I went into `virt-manager` and created a network called "default" and used NAT for it.

```
sudo virt-install --virt-type kvm --name centos --ram 4096 \
  --disk /tmp/centos.qcow2,format=qcow2 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=rhel7 \
  --location=/tmp/CentOS-7-x86_64-Minimal-1511.iso
```

Now bring up `virt-manager` and attach to a console.

For the options, I've used the defaults unless otherwise noted here...

* Turn on the ethernet device
* Used default partitioning scheme.
* Set a root password, create user "centos" with password "Redhat01" and make them an administrator.

Detach the "CD-ROM" (lol), while you're at it, also disconnect the VHS player. Figure out the device for the CD-ROM, then reboot the instance.

```
virsh dumpxml centos | grep -i -A5 "cdrom"
virsh attach-disk --type cdrom --mode readonly centos "" hdb
# "Luckily" I also had to force reset mine at virt-manager (sarcasm intended)
virsh reboot centos
```

Login at the console, test that the networking works, and use `ip a` to figure out the IP address, then we can SSH in to make things easier.

```
[root@localhost tmp]# ssh root@192.168.100.238
```

Let's update everything with yum because that makes life better. 99.99999% chance you'll get a new kernel, so reboot for good measure, too.

```
yum update -y
reboot
```

Install ACPI for power management by OpenStack

```
yum install -y acpid
systemctl enable acpid
```

We're gonna need `cloud-init`, so let's get that up and configured.

```
yum install -y epel-release.noarch
yum install -y cloud-init

# Check what the default user is, mine said "centos" so I'm happy with that.
grep -A3 "default_user" /etc/cloud/cloud.cfg
```

Disable zeroconf

```
echo "NOZEROCONF=yes" >> /etc/sysconfig/network
```

Configure so that the nova console works.

Per the reference doc, "Edit the /etc/default/grub file and configure the `GRUB_CMDLINE_LINUX` option. Delete the `rhgb quiet` and add the `console=tty0 console=ttyS0,115200n8` to the option"

```
[root@localhost ~]# vi /etc/default/grub 
[root@localhost ~]# cat /etc/default/grub 
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap console=tty0 console=ttyS0,115200n8"
GRUB_DISABLE_RECOVERY="true"
```

Then enable that with:

```
grub2-mkconfig -o /boot/grub2/grub.cfg
```

BUT WAIT! THERE'S MORE! What did we originally do this all for? We wanted to enable NetworkManager. But guess what? In Minimal CentOS, it's already enabled, but, just to be sure, double check...

```
systemctl status NetworkManager
```

And poweroff...

```
/sbin/shutdown -h now
```

Finally, let's cleanup. This will remove the MAC from having it spun up.

```
virt-sysprep -d centos
```

Now, remove the image...

```
virsh undefine centos
```

And you should be left with a centos image...

```
[root@localhost tmp]# ls -lathr centos.qcow2 
-rw-r--r-- 1 root root 1.5G Nov  3 11:59 centos.qcow2
```

Voila! Now send it where to your undercloud, if you're me.

```
[doug@localhost tmp]$ scp -F ~/.quickstart/ssh.config.ansible /tmp/centos.qcow2 stack@undercloud:/tmp/centos-custom.qcow2
```

Now you can upload it to glance

```
source ~/overcloudrc
glance image-create --name centos-custom --disk-format qcow2  --container-format bare --file /tmp/centos-custom.qcow2 --progress
internal_net=$(neutron net-list | awk ' /int/ {print $2;}')
```

And we'll boot an instance to test

```
nova boot --flavor m1.small --key-name atomic_key --nic net-id=$internal_net --image centos-custom centos1
```

Associate a floating IP (I used the openstack dashboard) and ssh to it

```
[stack@undercloud tmp]$ ssh -i ~/atomic_key.pem centos@192.168.24.107
```

And does it have network manager?

```
[centos@centos1 ~]$ sudo systemctl status NetworkManager | grep -i active
   Active: active (running) since Thu 2016-11-03 13:39:48 EDT; 1min 33s ago
```

Woooot! And there we have it.