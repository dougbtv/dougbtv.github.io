---
author: dougbtv
comments: true
date: 2017-05-12 14:21:01-05:00
layout: post
slug: kubernetes-from-source
title: Gleaming the Kube - Building Kubernetes from source
category: nfvpe
---

Time to bust out your kicktail skateboards Christian Slater fans, we're about to [gleam the kube](https://www.youtube.com/watch?v=-pYYZ6LdxWQ), or... really, fire up our terminals and build Kubernetes from source. Then we'll manually install what we built and configure it and get it running. Most of the time, you can just `yum install` or `docker run` the things you need, but, sometimes... That's just not enough when you're going to need some finer grain control. Today we're going to look at exactly how to build Kubernetes from source, and then how to deploy a working Kubernetes given the pieces therein. I base this tutorial on [using the official build instructions from the Kube docs](https://kubernetes.io/docs/getting-started-guides/binary_release/#building-from-source) to start. But, the reality is as much as it's easy to say the full instructions are `git clone` and `make release` -- that just won't cut the mustard. We'll need to do a couple ollies and pop-shove-its to really get it to go. Ready? Let's roll...

The goals here today are to:

* Build binaries from Kubernetes source code
* Install the binaries on a system
* Configure the Kubernetes system (for a single node)
* Run a pod to verify that it's all working.

## Requirements

This walk-through assumes a CentOS 7.3 machine, for one. And I use a VM to isolate this from my usual workstation environment. I wind up in this tutorial spinning up two boxes -- one to build Kubernetes, and one to deploy Kubernetes on. You can do it all on one host if you choose.

I setup my VMs often times using [this great article for spinning up centos cloud images](http://giovannitorres.me/create-a-linux-lab-on-kvm-using-cloud-images.html) but it relies only on having the default disk size for the CentOS cloud images, which is 8 gigs which isn't enough.

Definitely need a lot of memory, I had it bomb out at 8 gigs, so I bumped up to 16 gigs. 

Then -- you need a lot more disk than I had. I've been using VMs spun up with virsh to setup this environment, so I'll have to add some storage, which is outlined below. You need AT LEAST 20 gigs free. I'm not sure what it's doing to need that must disk, but, it needs it. I go with 24 gigs below, and outline how to add an extra disk for the `/var/lib/docker/` directory.

## Spinning up a VM and adding a disk

If you've got another machine to use, that's fine -- and you can skip this section. Make sure you have at least 24 gig of free disk space available. You can even potentially use your workstation, if you'd like.

Go ahead and spin up a VM, in my case I'm spinning up CentOS cloud images. So let's attach a spare disk, and we'll use it for `/var/lib/docker/`.

Spin up a VM however you like with virsh and let's create a spare disk image. I'm going to create one that's 24 gigs.

Looking at your **virtual machine host** Here's my running instance, called `buildkube`

```bash
$ virsh list | grep -Pi "( Id|buildkube|\-\-\-)"
 Id    Name                           State
----------------------------------------------------
 5     buildkube                      running
```

Create a spare disk:

```
$ dd if=/dev/zero of=/var/lib/libvirt/images/buildkube-docker.img bs=1M count=24576
```

Then we can attach it.

```bash
$ virsh attach-disk "buildkube" /var/lib/libvirt/images/buildkube-docker.img vdb --cache none
Disk attached successfully
```

Now, let's ssh to the **guest virtual machine** and we'll format and mount the disk.

From there we'll use fdisk to make a new partition

```
$ fdisk /dev/vdb
```

Choose:

* `n`: New Partition
* `p`: Primary partition
* Accept defaults for part number, first and last sector.
* `w`: Write the changes.
* `q`: quit

Now that you've got that set, let's format it and give it a label.

```
$ mkfs -t ext3 /dev/vdb1
$ e2label /dev/vdb1 /docker
```

Add it to the `fstab`, and mount it. Create an entry in `/etc/fstab` like so:

    /dev/vdb1   /var/lib/docker ext3    defaults    0   0

Now, create a directory to mount it and mount it.

```
$ mkdir -p /var/lib/docker
$ mount /var/lib/docker/
```

I then like to create a file and edit it to make sure things are OK.

Alright -- you should be good to go.

## Readying for install

Alright, first things first -- go ahead and do a `yum update -y` and install any utils you want (your editor, etc) -- and give 'er a reboot.

Let's install some utilities we're going to use; git & tmux.

```
$ sudo yum install -y tmux git
```

Install docker-ce [per the docs](https://docs.docker.com/engine/installation/linux/centos/#install-using-the-repository)

```
$ sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install docker-ce
$ sudo systemctl enable docker
$ sudo systemctl start docker
$ sudo docker -v
```

Now that we've got docker ready, we can start onto the builds (they use Docker, fwiw.)

## Kick off the build

Ok, get everything ready, clone a specific tag in this instance (with a shallow depth) and move into that dir and fire up tmux (cause it's gonna take a while)

```
$ git clone -b v1.6.3 --depth 1 https://github.com/kubernetes/kubernetes.git
$ cd kubernetes
$ tmux new -s build
```

Now let's choose a release style. If you want bring up the `Makefile` in an editor and surf around, the official docs recommend `make release`, but, in order to quicken this up (still takes a while, about 20 minutes on my machine), we'll do a `quick-release` (which doesn't cross-compile or run the full test suite)

```
$ make quick-release
```

You can now exit the tmux screen while you do other things (or... get yourself a coffee, that's what I'm going to do), you can do so with `ctrl+b` then `d`. (And you can return to it with `tmux a` -- assuming it's the only session, or `tmux a -t build` if there's multiple sessions). Should this fail -- read through and see what's going on. Mostly -- I recommend to begin with starting with a tagged release (master isn't always going to build, I've found), that way you know it should in theory have clean enough code to build, and then assuming that's OK check your disk and memory situation as it's rather stringent requirements there, I've found.

Alright, now... Back with your coffee and it's finished? Awesome, let's peek around and see what we created. It creates some tarballs and some binaries, so we can see where they are.

```
$ find . | grep gz
$ find . | grep -iP "(client|server).bin"
```

For now we're just concerned with those binaries, and we'll move 'em into the right place.

## Let's deploy it

I spun up another host with standard disk (8gig in my case) and I'm going to put the pieces together there.

Surprise! We need another `yum update -y` and a docker install. So let's perform that on this "deploy host"

```
$ sudo yum-config-manager --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
$ yum install -y docker-ce
$ systemctl enable docker --now
$ yum update -y 
$ reboot
```

### A place for everything, and everything in its place

Now, from your build host, scp some things... Let's try to make it as close to the rpm install as possible.

```
$ pwd
/home/centos/kubernetes
$ scp -i ~/.ssh/id_vms ./_output/release-stage/client/linux-amd64/kubernetes/client/bin/kubectl centos@192.168.122.147:~       
$ scp -i ~/.ssh/id_vms ./_output/release-stage/server/linux-amd64/kubernetes/server/bin/kubeadm centos@192.168.122.147:~
$ scp -i ~/.ssh/id_vms ./_output/release-stage/server/linux-amd64/kubernetes/server/bin/kubelet centos@192.168.122.147:~
$ scp -i ~/.ssh/id_vms ./build/debs/kubelet.service centos@192.168.122.147:~
$ scp -i ~/.ssh/id_vms ./build/debs/kubeadm-10.conf centos@192.168.122.147:~
```

Alright, that's everything but CNI, basically.

Now back to the deploy host, let's place the binaries and in the configs in the right places...

```bash

# Move binaries
[root@deploykube centos]# mv kubeadm /usr/bin/
[root@deploykube centos]# mv kubectl /usr/bin/
[root@deploykube centos]# mv kubelet /usr/bin/

# Move systemd unit and make directories
[root@deploykube centos]# mv kubelet.service /etc/systemd/system/kubelet.service
[root@deploykube centos]# mkdir -p /etc/kubernetes/manifests
[root@deploykube centos]# mkdir -p /etc/systemd/system/kubelet.service.d/
[root@deploykube centos]# mv kubeadm-10.conf /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Edit kubeadm config, with two lines from below
[root@deploykube centos]# vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

When you edit that `10-kubeadm.conf` add these two lines above the `ExecStart=` line:

```
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
```

### Install CNI binaries

Now, we gotta setup CNI. First, clone and build it (you need git and golang).

```
[root@deploykube centos]# yum install -y git golang
[root@deploykube centos]# git clone -b v0.5.2 --depth 1 https://github.com/containernetworking/cni.git
[root@deploykube centos]# cd cni/
[root@deploykube cni]# ./build.sh 

```

With that complete, we can now copy out the binary plugins.

```
[root@deploykube cni]# mkdir -p /opt/cni/bin
[root@deploykube cni]# cp bin/* /opt/cni/bin/
```

Ok, looking like... we're potentially close. Reload units and start & enable kubelet.

```
[root@deploykube cni]# systemctl daemon-reload
[root@deploykube cni]# systemctl enable kubelet --now
```

Now, let's try a kubeadm. We're going to aim to use flannel.

```
[root@deploykube cni]# kubeadm init --pod-network-cidr 10.244.0.0/16
```

Amazingly.... that completed for me on the first go, guess I did my homework! 

From that output you'll also get your join commands should you want to expand the cluster beyond this one node. You don't need to run this if you're just running a single node like me in this tutorial. The command will look like:

```
  kubeadm join --token 49cb93.48ac0d64e3f6ccf6 192.168.122.147:6443
```

Follow steps to use the cluster with kubectl, must be a regular non-root user. So we'll use centos in this case.

```
[centos@deploykube ~]$   sudo cp /etc/kubernetes/admin.conf $HOME/
[centos@deploykube ~]$   sudo chown $(id -u):$(id -g) $HOME/admin.conf
[centos@deploykube ~]$   export KUBECONFIG=$HOME/admin.conf
[centos@deploykube ~]$ kubectl get nodes
NAME                       STATUS     AGE       VERSION
deploykube.example.local   NotReady   52s       v1.6.3
```

And let's watch that for a while... Get yourself a coffee here for a moment.

```
[centos@deploykube ~]$ watch -n1 kubectl get nodes
```

### Install pod networking, flannel here.

Tricked ya! Hope you didn't get coffee already. Wait, that won't be ready until we have a pod network. So let's add that.

I put the yamls in a gist, so we can curl 'em like so:

```
$ curl https://gist.githubusercontent.com/dougbtv/a6065c316019642ecc1706d6e785a037/raw/16554948e306359090c3d52c3c7b0bcffea2e450/flannel-rbac.yaml > flannel-rbac.yaml
$ curl https://gist.githubusercontent.com/dougbtv/a6065c316019642ecc1706d6e785a037/raw/16554948e306359090c3d52c3c7b0bcffea2e450/flannel.yaml > flannel.yaml
```

And apply those...

```
[centos@deploykube ~]$ kubectl apply -f flannel-rbac.yaml 
clusterrole "flannel" created
clusterrolebinding "flannel" created
[centos@deploykube ~]$ kubectl apply -f flannel.yaml 
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
```

Now.... you should have a ready node.

```
[centos@deploykube ~]$ kubectl get nodes
NAME                       STATUS    AGE       VERSION
deploykube.example.local   Ready     21m       v1.6.3
```

## Run a pod to verify we can actually use Kubernetes

Alright, so... That's good, now, can we run a pod? Let's use my favorite example, 2 nginx pods via a replication controller.

```
[centos@deploykube ~]$ cat nginx.yaml 
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

And watch until they come up... You should have a couple...

```
[centos@deploykube ~]$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
nginx-5hmxs   1/1       Running   0          41s
nginx-m42cz   1/1       Running   0          41s
```

And let's curl something from them...

```
[centos@deploykube ~]$ kubectl describe pod nginx-5hmxs | grep -P "^IP"
IP:     10.244.0.3
[centos@deploykube ~]$ curl -s 10.244.0.3 | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

Word! That's a running kubernetes from source.

---

## Some further inspection of the built pieces

What follow are some notes of mine while I got all the pieces together to make the build work. If you're interested in some of the details, it might be worthwhile reading, however, it's somewhat raw notes, I didn't overly groom it before posting it.

### The `kubernetes.tar.gz` tarball

Let's take a look at the `kubernetes.tar.gz`, that sounds interesting. It kind of is. It looks like a lot of setup goods.

```
$ cd _output/release-tars/
$ cp kubernetes.tar.gz /tmp/
```

And I'm in the `./cluster/centos` and doing `./build.sh` -- failed. So I started reading about it, and it points at [this old doc for a centos cluster](http://kubernetes.github.io/docs/getting-started-guides/centos/centos_manual_config/) which then says "Oh yeah, that's deprecated." and then pointed to the [contemporary kubeadm install method](https://kubernetes.io/docs/getting-started-guides/kubeadm/) (which I've been using in kube-ansible).

### So, how are we going to install it so we can use it?

Kubernetes isn't terribly bad to deal with regarding deps, why? Golang. Makes it simple for us.

So let's see how a rpm install looks after all, we'll use that as our basis for installing and configuring kube.

Usually in kube-ansible, I install these rpms

```
- kubelet
- kubeadm
- kubectl
- kubernetes-cni
```

Here's what results from these.

```
[root@kubecni-master centos]# rpm -qa | grep -i kub
kubernetes-cni-0.5.1-0.x86_64
kubectl-1.6.2-0.x86_64
kubeadm-1.6.2-0.x86_64
kubelet-1.6.2-0.x86_64

[root@kubecni-master centos]# rpm -qa --list kubernetes-cni
/opt/cni
/opt/cni/bin
/opt/cni/bin/bridge
/opt/cni/bin/cnitool
/opt/cni/bin/dhcp
/opt/cni/bin/flannel
/opt/cni/bin/host-local
/opt/cni/bin/ipvlan
/opt/cni/bin/loopback
/opt/cni/bin/macvlan
/opt/cni/bin/noop
/opt/cni/bin/ptp
/opt/cni/bin/tuning

[root@kubecni-master centos]# rpm -qa --list kubectl
/usr/bin/kubectl

[root@kubecni-master centos]# rpm -qa --list kubeadm
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
/usr/bin/kubeadm

[root@kubecni-master centos]# rpm -qa --list kubelet
/etc/kubernetes/manifests
/etc/systemd/system/kubelet.service
/usr/bin/kubelet
```

**So the binaries, that makes sense, but where do the etc pieces come from?**

So it appears that in the clone, the 

* `./build/debs/kubelet.service` == `/etc/systemd/system/kubelet.service`
* `./build/debs/kubeadm-10.conf` ~= `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`

The kubeadm.conf is... similar, but not exactly. I modify it to add a cgroup driver, and there's also two additional lines from the RPM

```
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
```

Didn't find it in the git clone with `grep -Prin "cgroup-driver"`

I also realize in my playbooks I do this...

```yaml
- name: Add custom kubadm.conf ExecStart
  lineinfile:
    dest: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    regexp: 'systemd$'
    line: 'ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_EXTRA_ARGS --cgroup-driver=systemd'
```

So I accounted for that as well.