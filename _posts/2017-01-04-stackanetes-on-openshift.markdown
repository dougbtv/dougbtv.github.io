---
author: dougbtv
comments: true
date: 2017-01-04 16:00:00-05:00
layout: post
slug: stackanetes-on-openshift
title: Running Stackanetes on Openshift
category: nfvpe
---

[Stackanetes](https://github.com/stackanetes/stackanetes) is an open-source project that aims to run OpenStack on top of Kubernetes. Today we're going to use a project that I created that uses ansible plays to setup Stackanetes on Openshift, [openshift-stackanetes](https://github.com/dougbtv/openshift-stackanetes). We'll use an all-in-one server approach to setting up openshift in this article to simplify that aspect, and later provide playbooks to launch Stackanetes with a cluster and focus on HA requirements in the future.

If you're itching to get into the walk-through, head yourself down to the [requirements section](#requirements) and you can get hopping. Otherwise, we'll start out with an intro and overview of what's involved to get the components together in order to make all that good stuff down in that section work in concert.

During this year's OpenStack summit, and [announced on](https://coreos.com/blog/announcing-stackanetes-technical-preview.html) October 26th 2016, Stackanetes was demonstrated as a [technical preview](https://techcrunch.com/2016/10/26/coreos-launches-its-openstack-on-kubernetes-project-as-a-technical-preview/). Up until this point, I don't believe it has been documented as being run on OpenShift. I wouldn't be able to document this myself if it weren't for the rather gracious assistance of the crew from the CoreOS project and the Stackanetes as they helped me through [this issue](https://github.com/stackanetes/stackanetes/issues/144) on GitHub. Big thanks go to [ss7pro](https://github.com/ss7pro), [PAStheLod](https://github.com/PAStheLoD), [ant31](https://github.com/ant31), and [Quentin-M](https://github.com/Quentin-M). Really appreciated the help crew, big time!

On terminology -- while the Tech Crunch article considers the name Stackanetes unfortunate, I disagree -- I like the name. It kind of rolls off the tongue. Also if you say it fast enough, someone might say "[Gesundheit!](https://en.wikipedia.org/wiki/Responses_to_sneezing)" after you say it. Also, theoretically using the construst of i18n (internationalization) or better yet, k8s (Kubernetes), you could also say this is s9s (stackanetes), which I'd use in my commit messages and what not because... It's a bit of typing! You might see s9s here and again in this article, too. Also, you might hear me say "OpenShift" a huge number of times -- I really mean "OpenShift Origin" whenever I say it.

---

## Scope of this walk-through

Primarily we'll focus on using an all-in-one OpenShift instance, that is one that uses the `oc cluster up` command to run OpenShift all on a single host, as outlined in the [local cluster management documentation](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md). My "[openshift on openstack in easy mode](http://dougbtv.com/nfvpe/2016/10/31/openshift-on-openstack-easy-mode/)" goes into some of those details as well. However, the playbooks will take care of this setup for you in this case.

Things we do cover:

* Getting OpenShift up (all-in-one style, or what I like to call "easy mode")
* Spinning up a [KPM registry](https://github.com/coreos/kpm)
* Setting up proper permissions for Stackanetes to run under OpenShift
* Getting Stackanetes running in OpenShift

Things we don't cover:

* High availability (hopefully we'll look at this in a further article)
* For now, tenant / external networking, we'll just run OpenStack clouds instances in their own isolated network.
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

Now that you've got that good to go