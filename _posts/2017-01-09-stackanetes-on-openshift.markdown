---
author: dougbtv
comments: true
date: 2017-01-09 08:20:01-05:00
layout: post
slug: stackanetes-on-openshift
title: Running Stackanetes on Openshift
---

[Stackanetes](https://github.com/stackanetes/stackanetes) is an open-source project that aims to run OpenStack on top of Kubernetes. Today we're going to use a project that I created that uses ansible plays to setup Stackanetes on Openshift, [openshift-stackanetes](https://github.com/dougbtv/openshift-stackanetes). We'll use an all-in-one server approach to setting up openshift in this article to simplify that aspect, and later provide playbooks to launch Stackanetes with a cluster and focus on HA requirements in the future.

If you're itching to get into the walk-through, head yourself down to the [requirements section](#requirements) and you can get hopping. Otherwise, we'll start out with an intro and overview of what's involved to get the components together in order to make all that good stuff down in that section work in concert.

During this year's OpenStack summit, and [announced on](https://coreos.com/blog/announcing-stackanetes-technical-preview.html) October 26th 2016, Stackanetes was demonstrated as a [technical preview](https://techcrunch.com/2016/10/26/coreos-launches-its-openstack-on-kubernetes-project-as-a-technical-preview/). Up until this point, I don't believe it has been documented as being run on OpenShift. I wouldn't be able to document this myself if it weren't for the rather gracious assistance of the crew from the CoreOS project and the Stackanetes as they helped me through [this issue](https://github.com/stackanetes/stackanetes/issues/144) on GitHub. Big thanks go to [ss7pro](https://github.com/ss7pro), [PAStheLod](https://github.com/PAStheLoD), [ant31](https://github.com/ant31), and [Quentin-M](https://github.com/Quentin-M). Really appreciated the help crew, big time!

On terminology -- while the Tech Crunch article considers the name Stackanetes unfortunate, I disagree -- I like the name. It kind of rolls off the tongue. Also if you say it fast enough, someone might say "[Gesundheit!](https://en.wikipedia.org/wiki/Responses_to_sneezing)" after you say it. Also, theoretically using the construst of i18n (internationalization) or better yet, k8s (Kubernetes), you could also say this is s9s (stackanetes), which I'd use in my commit messages and what not because... It's a bit of typing! You might see s9s here and again in this article, too. Also, you might hear me say "OpenShift" a huge number of times -- I really mean "OpenShift Origin" whenever I say it.

---

## Scope of this walk-through

First thing's first -- [openshift-stackanetes](https://github.com/dougbtv/openshift-stackanetes) is the project we'll focus on to use to spin up Stackanetes on Openshift, it is a series of Ansible roles to help us accomplish getting Stackanetes on OpenShift.

Primarily we'll focus on using an all-in-one OpenShift instance, that is one that uses the `oc cluster up` command to run OpenShift all on a single host, as outlined in the [local cluster management documentation](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md). My "[openshift on openstack in easy mode](http://dougbtv.com/nfvpe/2016/10/31/openshift-on-openstack-easy-mode/)" goes into some of those details as well. However, the playbooks will take care of this setup for you in this case.

Things we do cover:

* Getting OpenShift up (all-in-one style, or what I like to call "easy mode")
* Spinning up a [KPM registry](https://github.com/coreos/kpm)
* Setting up proper permissions for Stackanetes to run under OpenShift
* Getting Stackanetes running in OpenShift

Things we don't cover:

* High availability (hopefully we'll look at this in a further article)
* For now, tenant / external networking, we'll just run OpenStack clouds instances in their own isolated network. (This is kind of a project on its own)
* In depth usage of OpenStack -- we'll just do enough to get some cloud instances up
* Spinning up Ceph
* A sane way of exposing DNS externally (we'll just use a hosts file for our client machines outside of the s9s box)
* Details of how to navigate OpenShift, surf this blog for some basics if you need them.
* Changing out the container runtime (e.g. using rkt, we just use Docker this time around)
* [Ansible installation](http://docs.ansible.com/ansible/intro_installation.html) and basic usage, we will however give you all the ansible commands to run this playbook.

---

## Considerations of using Stackanetes on OpenShift

Some of the primary considerations I had to overcome for using Stackanetes on OpenShift is managing the SCCs ([security context constraints](https://docs.openshift.org/latest/architecture/additional_concepts/authorization.html#security-context-constraints)).

I'm not positive that the SCCs I have defined herein in are ideal. In some ways, I can point out that they are insufficient in a few ways. However, my initial focus has been to get to Stackanetes to run properly.  

---

## Components of openshift-stackanetes

So, first off there's a lot of components of Stackanetes, especially the veritable cornucopia of pieces that comprise OpenStack. If you're interested in those, you might want to check out the [Wikipedia article on OpenStack](https://en.wikipedia.org/wiki/OpenStack) which has a fairly comprehensive list.

One very interesting part of what stackanetes is that it leverages [KPM registry](https://github.com/coreos/kpm).

KPM is described as "a tool to deploy and manage application stacks on Kubernetes". I like to think of it as "k8s package manager", and while never exactly branded that way, that makes sense to me. In my own words -- it's a way to take the definition YAML files you'd use to build k8s resources and parameterize them, and then store them in a registry so that you can access them later. In a word: brilliant.

Something I did in the process of creating openshift-stackanetes was to create an [Ansible-Galaxy role for KPM on Centos](https://galaxy.ansible.com/dougbtv/kpm-install/) to get a contemporary revision of kpm client running on CentOS, it's included in the openshift-stackanetes ansible project as a requirement.

Another really great component of s9s is that they've gone ahead and implemented [Traefik](https://github.com/containous/traefik) -- which is a fairly amazing "modern reverse proxy" (Traefik's words). This doles out the HTTP requests to the proper services.

Let's give a quick sweeping overview of the roles as ran by the openshift-stackanetes playbooks:


* `docker-install` installs the latest Docker from the official Docker RPM repos for CentOS.
* `dougbtv.kpm-install` installs the KPM client to the OpenShift host machine.
* `openshift-install` preps the machine with the deps to get OpenShift up and running.
* `openshift-up` generally runs the `oc cluster up` command.
* `kpm-registry` creates a namespace for the KPM registry and spins up the pods for it.
* `openshift-aio-dns-hack` is my "all-in-one" OpenShift DNS hack.
* `stackanetes-configure` preps the pieces to go into the kpm registry for stackanetes and spins up the pods in their own namespace.
* `stackanetes-routing` creates routes in OpenShift for the stackanetes services that we need to expose

---

## <a name="requirements"></a> Requirements

* A machine with CentOS 7.3 installed
* 50 gig HDD minimum (64+ gigs recommended)
* 12 gigs of RAM minimum
* 4 cores recommended
* Networking pre-configured to your liking
* SSH keys to root on this machine from a client machine
* A client machine with git and [ansible installed](http://docs.ansible.com/ansible/intro_installation.html).

You can use a virtual machine or bare metal, it's your choice. I do highly recommend doubling all those above requirements though, and using a bare metal machine as your experience will 

If you use a virtual machine you'll need to make sure that you have nested virtualization passthrough. I was able to make this work, and while I won't go into super details here, the gist of what I did was to check if there were virtual machine extensions on the host, and also the guest. You'll node I was using an AMD machine.

```
# To check if you have virtual machine extensions (on host and guest)
$ cat /proc/cpuinfo | grep -Pi "(vmx|svm)"

# Then check that you have nesting enabled
$ cat /sys/module/kvm_amd/parameters/nested
1
```

And then I needed to use the `host-passthrough` CPU mode to get it to work.

```
$ virsh dumpxml stackanetes | grep -i pass
  <cpu mode='host-passthrough'/>
```

All that said, I still recommend the bare metal machine, and my notes were double checked against bare metal... I think your experience will be improved, but I realize that isn't always a convenient option.

---

## Let's run some playbooks!

So, we're assuming that you've got your CentOS 7.3 machine up, you know its IP address and you have SSH keys to the root user. (Don't like the root user? I don't really, feel free to contribute updates to the playbooks to properly use `become`!)

### git clone and basic ansible setup

First things first, make sure you have ansible installed on your client machine, and then we'll clone the repo.

```
$ git clone https://github.com/dougbtv/openshift-stackanetes.git
$ cd openshift-stackanetes
```

Now that we have it installed, let's go ahead and modify the `inventory` file in the root directory. In theory, all you should need to do is change the IP address there to the CentOS OpenShift host machine.

It looks about like this:

```
$ cat inventory && echo
stackanetes ansible_ssh_host=192.168.1.100 ansible_ssh_user=root

[allinone]
stackanetes
```

### Ansible variable setup

Now that you've got that good to go, you can modify some of our local variables, check out the `vars/main.yml` file to see the variables you can change. 

There's two important variables you may need to change:

* `facter_ipaddress`
* `minion_interface_name`

Firstly `facter_ipaddress` variable. This is important as the value of this determines how we're going to find your IP address. By default it's set to `ipaddress`. If you're unsure what to put here, go ahead and install facter and check out which value returns the IP address you'd like to use for external access ot the machine.

```
[root@labstackanetes ~]# yum install -y epel-release
[root@labstackanetes ~]# yum install -y facter
[root@labstackanetes ~]# facter | grep -i ipaddress
ipaddress => 192.168.1.100
ipaddress_enp1s0f1 => 192.168.1.100
ipaddress_lo => 127.0.0.1
```

In this case, you'll see that either `ipaddress` or `ipaddress_enp1s0f1` look like valid choices -- however the `ipaddress` isn't reliable, so choose one based on your NIC.

Next the `minion_interface_name`, additionally important because this is the interface we're going to tell Stackanetes to use for networking for the pods it deploys. This should generally be the same interface that the above ip address came from.

You can either edit the `./vars/main.yml` file or you can pass them in as extra vars e.g. `--extra-vars "facter_ipaddress=ipaddress_enp1s0f1 minion_interface_name=enp1s0f1"`

### Let's run that playbook!

Now that you're setup, you should be able to run the playbook...

The default way you'd run the playbook is with...

```
$ ansible-playbook -i inventory all-in-one.yml
```

Or if you're specifying the `--extra-vars`, insert that before the yaml filename.

### If everything has gone well!

It likely may have! If everything has gone as planned, there should be some output that will help you get going...

It should list:

* The location of the openshift dashboard, e.g. `https://yourip:8443`
* The location of the KPM registry (a cluster.local URL)
* A series of lines representing a `/etc/hosts` file to put on your client machine.

You should be able to check out the OpenShift dashboard (cockpit) and take a little peek around to see what has happened.

### Possible "gotchyas" and troubleshooting

First thing's first -- you can log into the openshift host and issue:

```
oc projects
oc project openstack
oc get pods
```

And see if any pods are in error. 

The biggest possibility of what has gone wrong is that etcd in the kpm package didn't come up properly. This happens intermittently to me, and I haven't debugged it, nor opened up an issue with the KPM folks (Unsure if it's how they instantiate etcd or etcd itself, I do know however that spinning up an etcd cluster can be a precarious thing, so, it happens.)

In this case that this happens, go ahead and delete the KPM namespace and run the playbook again, e.g.

```
# Change away from the kpm project in case you're on it
oc project default
# Delete the project / namespace
oc project delete kpm
# List the projects to see if it's gone before you re-run
oc projects
```

---

## Let's access OpenStack!

Alright! You got this far, nice work... You're fairly brave if you made it this far. I've been having good luck, but, I still appreciate your bravado!

First up -- did you make an `/etc/hosts` file on your local machine? We're not worrying about external DNS yet, so you'll have to do that, it will have entries that look somewhat similar to this but has your IP address of your OpenShift host:

```
192.168.1.100 identity.openstack.cluster
192.168.1.100 horizon.openstack.cluster
192.168.1.100 image.openstack.cluster
192.168.1.100 network.openstack.cluster
192.168.1.100 volume.openstack.cluster
192.168.1.100 compute.openstack.cluster
192.168.1.100 novnc.compute.openstack.cluster
192.168.1.100 search.openstack.cluster
```

So, you can access Horizon (the OpenStack dashboard) by pointing your browser at:

```
http://horizon.openstack.cluster
```

Great, now just login with username "admin" and password "password", aka SuperSecure(TM).

Surf around that until you're satisfied that the GUI isn't powerful enough and you now need to hit up the command line ;)

---

## Using the openstack client

Go ahead and SSH into the OpenShift host machine, and in root's home directory you'll find that there's a `stackanetesrc` file available there. It's based on the `/usr/src/stackanetes/env_openstack.sh` file that comes in the Stackanetes git clone.

So you can use it like so and get kickin'

```
[root@labstackanetes ~]# source ~/stackanetesrc 
[root@labstackanetes ~]# openstack hypervisor list
+----+----------------------------+-----------------+---------------+-------+
| ID | Hypervisor Hostname        | Hypervisor Type | Host IP       | State |
+----+----------------------------+-----------------+---------------+-------+
|  1 | identity.openstack.cluster | QEMU            | 192.168.1.100 | down  |
|  2 | labstackanetes             | QEMU            | 192.168.1.100 | up    |
+----+----------------------------+-----------------+---------------+-------+
```

### So how about a cloud instance!?!?!!!

Alright, now that we've sourced our run commands, we can go ahead and configure up our OpenStack so we can run some instances. There's a handy file for a suite demo commands to spin up some instances packaged in Stackanetes itself, my demo here is based on the same. You can find that configuration @ `/usr/src/stackanetes/demo_openstack.sh`.

First up, we download the infamous cirros & upload it to glance.

```
$ source ~/stackanetesrc 
$ curl -o /tmp/cirros.qcow2 http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ openstack image create --disk-format qcow2  --container-format bare  --file /tmp/cirros.qcow2 cirros
```

Now let's create our networks

```
# External Net
$ openstack network create ext-net --external --provider-physical-network physnet1 --provider-network-type flat
$ openstack subnet create ext-subnet --no-dhcp --allocation-pool start=172.17.0.25,end=172.17.0.50 --network=ext-net --subnet-range 172.17.0.0/24 --gateway 172.17.0.1

# Internal Net
$ openstack network create int
$ openstack subnet create int-subnet --allocation-pool start=30.0.0.2,end=30.0.0.254 --network int --subnet-range 30.0.0.0/24 --gateway 30.0.0.1 --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4
$ openstack router create demo-router
$ neutron router-interface-add demo-router $(openstack subnet show int-subnet -c id -f value)
$ neutron router-gateway-set demo-router ext-net
```

Alright, now let's at least add a flavor.

```
$ openstack flavor create --public m1.tiny --ram 512 --disk 1 --vcpus 1
```

And a security group

```
$ openstack security group rule create default --protocol icmp
$ openstack security group rule create default --protocol tcp --dst-port 22
```

...Drum roll please. Here comes an instance! 

```
openstack server create cirros1 \
  --image $(openstack image show cirros -c id -f value) \
  --flavor $(openstack flavor show m1.tiny -c id -f value) \
  --nic net-id=$(openstack network show int -c id -f value)
```

Check that it hasn't errored out with a `nova list`, and then give it a floating IP.

```
# This should come packaged with a few new deprecation warnings.
$ openstack ip floating add $(openstack ip floating create ext-net -c floating_ip_address -f value) cirros1
```

### Let's do something with it!

So, you want to SSH into it? Well... Not yet. Go ahead and use Horizon to access the machine and then console into it, and ping the gateway, `30.0.0.1` in this example, there you go! You did something with it, and over the network.

Currently, I haven't got the provider network working, just a small isolated tenant network. So, we're saving that for next time. We didn't want to spoil all the fun for now, right!?

## Diagnosing a failed Nova instance creation.

So, the Nova instance didn't spin up, huh? There's a few reasons for that. To figure out the reason, first do a 

```
nova list
nova show $failed_uuid
```

That will likely give you a whole lot of nothing more than probably a "no valid host found". Which is essentially, nothing. So you're going to want to look at the Nova compute logs. We can get those with `kubectl` or the `oc` commands.

```
# Make sure you're on the openstack project
oc projects
# Change to that project
oc project openstack
# List the pods to find the "nova-compute" pod
oc get pods
# Get the logs for that pod
oc logs nova-compute-3298216887-sriaa | tail -n 10
```

Or in short.

```
$ oc logs $(oc get pods | grep compute | awk '{print $1}') | tail -n 50
```

Now you should be able to see something.

A few things that have happened to me intermittently.

1. You've sized your cluster wrong, or you're using a virtual container host, and it doesn't have nested virtualization. There might not be enough ram or processors for the instance, even though we're using a pretty darn micro instance here.

2. Something busted with openvswitch

I'd get an error like:

```
ovs-vsctl: unix:/var/run/openvswitch/db.sock: database connection failed (Protocol error)
```

So what I would do is delete the neutron-openvswitch pod, and it'd automatically deploy again and usually that'd do the trick.

3. One time I had a bad glance image, I just deleted it and uploaded to glance again, I lost the notes for this error but it was something along the lines of "writing to a .part file" errored out.


