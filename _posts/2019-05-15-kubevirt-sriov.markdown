---
author: dougbtv
comments: true
date: 2018-05-15 13:00:00-05:00
layout: post
slug: kubevirt-sriov
title: High Performance Networking with KubeVirt - SR-IOV device plugin to the rescue!
category: nfvpe
---

If you've got workloads that live in VMs, and you want to get them into your Kubernetes environment (because, I don't wish maintaining two platforms even on the worst of the supervillains!) -- you might also have networking workloads that require you to really push some performance.... KubeVirt with SR-IOV device plugin might be just the hero you need to save the day. Not all heros wear capes, sometimes those heroes just wear a t-shirt with a KubeVirt logo that they got at Kubecon. Today we'll spin up KubeVirt with SR-IOV device plugin and we'll run a VoIP workload on it, so jump into a phonebooth, change into your Kubevirt t-shirt and fire up a terminal!

I'll be giving a talk at Kubecon EU 2019 in Barcelona titled [High Performance Networking with KubeVirt](https://kccnceu19.sched.com/event/MPcw/high-performance-networking-with-kubevirt-doug-smith-red-hat-abdul-halim-intel). Presenting with me is the guy with the best Yoda drawing on all of GitHub, [Abdul Halim](https://github.com/ahalim-intel) from Intel. and I'll give a demonstration of what's going on here in this article, and this material will be provided to attendees too so that they can follow the bouncing ball and get the same demo working in their environment.

Part of the talk is this [recorded demo on YouTube](https://www.youtube.com/watch?v=Xzl9qQvaWtA&feature=youtu.be). It'll give you a preview of all that we're about to do here in this article. Granted this recorded demo does skip over some of the most interesting configuration, but, shows the results. We'll cover all the details herein to get you to the same point.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Xzl9qQvaWtA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

We'll look at spinning up KubeVirt, with SR-IOV capabilities. We'll walk through what the physical installation and driver setup looks like, we'll fire up KubeVirt, spin up VMs running in Kube, and then we'll put our VoIP workload (using [Asterisk](https://www.asterisk.org/)) in those pods -- which isn't complete until we terminate a phone call over a SIP trunk! The only thing that's on you is to install Kubernetes (but, I'll have pointers to get you started there, too). Just a quick note that I'm just using Asterisk as an example of a VoIP workload, it's definitely **NOT** limited to running in a VM, it [also works well in a container](https://github.com/dougbtv/docker-asterisk), even as a [containerized VNF](https://github.com/redhat-nfvpe/vnf-asterisk). You might be getting the point that I love Asterisk! (Shameless plugin, it's a great open source telephony solution!)

So -- why VMs? The thing is, maybe you're stuck with them. Maybe it's how your vendor shipped the software you bought and deploy. Maybe the management of the application is steeped in the history of it being virtualized. Maybe your software has legacies that simply just can't be easily re-written into something that's containerized. Maybe you like having pets (I don't always love pets in my production deployments -- but, I do love my cats Juniper & Otto, who I trained using know-how from [The Trainable Cat](https://www.goodreads.com/book/show/29101479-the-trainable-cat)! ...Mostly I just trained them to come inside on command as they're indoor-outdoor cats.)

Something really cool about the KubeVirt ecosystem is that it *REALLY* leverages some other hereos in the open source community. A good hero works well in a team for sure. In this case KubeVirt leverages [Multus CNI](http://multus-cni.io) which enables us to connect multiple network interfaces to pods (which also means VMs in the case of KubeVirt!), and we also use the [SR-IOV Device Plugin](https://github.com/intel/sriov-network-device-plugin) -- this plugin gives the Kubernetes scheduler awareness of which limited resources on our worker nodes have been exhausted -- specifically which SR-IOV virtual functions (VFs) have been used up, this way we schedule workloads on machines that have sufficient resources.

I'd like to send a HUGE thanks to [Booxter](https://github.com/booxter) -- Ihar from the KubeVirt team at Red Hat helped me get all of this going, and I could not have gotten nearly as far as I did without his help. Also thanks to [SchSeba](https://github.com/SchSeba) & [Phoracek](https://github.com/phoracek), too!

## Requirements

Not a ton of requirements, I think the heaviest two here is that you're going to need:

* Some experience with Kubernetes (you know how to use `kubectl` for some basic stuff, at least), and a way to install Kubernetes.
* SR-IOV capable devices on bare metal machines (and make them part of the Kubernetes cluster that you create)

I'm not going to cover the Kubernetes install here, I have some other material I will share with you on how to do so, though.

In my case, I spun up a cluster with [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/). Additionally, I also used my [kube-ansible playbooks](https://github.com/redhat-nfvpe/kube-ansible). If you'd like to use those playbooks, I also have another [blog article on how to use kube-ansible](http://dougbtv.com/nfvpe/2018/03/21/kubernetes-on-centos/).

### Install a "default network"

Once you have Kubernetes installed -- you're going to need to have some CNI plugin installed to act as the default network for your cluster. This will provide network connectivity between pods in the regular old fashioned way that you're used to. Why am I calling it the "default network", you ask? Because we're going to add additional network interfaces and attachments to other networks on top of this.

I used Flannel, and installed it like so:

```
$ curl https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml > flannel.yml
$ kubectl apply -f flannel.yml 
```

When it's installed you should see all nodes in a "ready" state when you issue `kubectl get nodes`.

## SR-IOV Setup

Primarily, I followed the [KubeVirt docs for SR-IOV setup](https://github.com/kubevirt/kubevirt/blob/master/docs/sriov.md). In my opinion, this is maybe the biggest adventure in this whole process -- mostly because depending on what SR-IOV hardware you have, and what mobo & CPU you have, etc... It might require you to have to dig deeply into your BIOS and figure out what to enable.

Mostly -- I will leave this adventure to you, but, I will give you a quick overview of how it went on my equipment.

It's a little like making a witch's brew, "Less eye of newt, more hair of frog... nope. Ok let's try that again, `blackcat_iommu=no ravensbreath_pci=on`"

Or as my co-worker Anton Ivanov said:

> It's just like that old joke about SCSI.
How many places do you terminate a SCSI cable?
Three. Once on each end and a black goat with a silver knife at full moon in the middle

Mostly, I first had to modify my kernel parameters, so, I added an extra `menuentry` in my `/etc/grub2.cfg`, and set it as the default with `grubby --set-default-index=0`, and made sure my `linux` line included:

```
amd_iommu=on pci=realloc
```

Make sure to do this on each node in your cluster that has SR-IOV hardware.

Note that I was using an AMD based motherboard and CPU, so you might have `intel_iommu=on` if you're using Intel, and the [KubeVirt docs](https://github.com/kubevirt/kubevirt/blob/master/docs/sriov.md) suggest a couple other parameters you can try.

If you need more help with Grub configurations, the [Fedora docs on working with the GRUB2 bootloader](https://docs.fedoraproject.org/en-US/fedora/rawhide/system-administrators-guide/kernel-module-driver-configuration/Working_with_the_GRUB_2_Boot_Loader/) are very helpful.

Then, in my BIOS I had to enable a number of things, I had to make sure SR-IOV support was on, as well as enabling IOMMU, and PCIe ARI Support.

After I had that up, I was able to find the VFs like so:

```
$ find /sys -name *vfs*
```

And then chose a `sriov_totalvfs` and echo that number into the `sriov_numvfs`:

```
$ cat /sys/devices/pci0000:00/0000:00:03.2/0000:2f:00.2/sriov_totalvfs
$ echo 32 > /sys/devices/pci0000:00/0000:00:03.2/0000:2f:00.2/sriov_numvfs
```

If it errors out, you might get a hint from following your journal, that is with `journalctl -f` and see if it gives you any hints. I almost thought I was going to have to modify my BIOS (gulp!), I had found [this Reddit thread](https://www.reddit.com/r/VFIO/comments/8pv6h7/sriov_not_functioning_on_asus_x370pro_board_w/), but, luckily it never got that far for me. It took me a few iterations at fixing my Kernel parameters and finding all the hidden bits in my BIOS, but... With patience I got there.

...Last but not least, make sure your physical ports on your SR-IOV card are connected to something. I had forgotten to connect mine initially and I couldn't get SR-IOV capable interfaces in my VMs to come up. So, back to our roots -- check layer 1!

### Make sure to `modprobe vfio-pci`

Make sure you have the `vfio-pci` kernel module loaded...

I did:

```
# modprobe vfio-pci
```

And then verified it with:

```
# lsmod | grep -i vfio
```

And then I added `vfio-pci` to `/etc/modules`

## KubeVirt installation

First we install the cluster-network-addons, this will install Multus CNI, and the SR-IOV device plugin.

Before we get any further, let's open the SR-IOV feature gate. So, on your machine where you use `kubectl`, issue:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config
  namespace: kubevirt
  labels:
    kubevirt.io: ""
data:
  feature-gates: "SRIOV"
EOF
```

It's assumed you'd generally do this on the master, or, wherever you run kubectl from.

Let's follow the [add-on operator deployment](https://github.com/kubevirt/cluster-network-addons-operator/#deployment)...

```
kubectl apply -f https://raw.githubusercontent.com/kubevirt/cluster-network-addons-operator/master/manifests/cluster-network-addons/0.7.0/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubevirt/cluster-network-addons-operator/master/manifests/cluster-network-addons/0.7.0/network-addons-config.crd.yaml
kubectl apply -f https://raw.githubusercontent.com/kubevirt/cluster-network-addons-operator/master/manifests/cluster-network-addons/0.7.0/operator.yaml
```

And we make an example custom resource...

```
kubectl apply -f https://raw.githubusercontent.com/kubevirt/cluster-network-addons-operator/master/manifests/cluster-network-addons/0.7.0/network-addons-config-example.cr.yaml
```

Watch for it all to come up...

```
$ watch -n1 kubectl get pods --all-namespaces -o wide
```

You can also use this wait condition...

```
$ kubectl wait networkaddonsconfig cluster --for condition=Ready
```

### Install the KubeVirt operator

Next we'll follow instructions from the [KubeVirt docs for installing the KubeVirt operator](https://kubevirt.io/user-guide/docs/latest/administration/intro.html#cluster-side-add-on-deployment). In this case we'll follow the "#2" instructions here for the "Alternative flow (aka Operator flow)".

It was suggested to me to use the latest version, as of this writing on the [KubeVirt releases](https://github.com/kubevirt/kubevirt/releases/) it's shown to be v0.17.0. 

```
$ export VERSION=v0.17.0
$ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt-operator.yaml
$ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt-cr.yaml
```

Watch the pods to be ready, `kubectl get pods` and all that good stuff.

Then we wait for this to be readied up...

```
$ kubectl wait kv kubevirt --for condition=Ready
```

(Mine never became ready?)

```
[centos@kube-nonetwork-master ~]$ kubectl wait kv kubevirt --for condition=Ready
Error from server (NotFound): kubevirts.kubevirt.io "kubevirt" not found
```

### Install `virtctl`

```
$ wget https://github.com/kubevirt/kubevirt/releases/download/v0.17.0/virtctl-v0.17.0-linux-amd64
$ chmod +x virtctl-v0.17.0-linux-amd64
$ sudo mv virtctl-v0.17.0-linux-amd64 /usr/bin/virtctl
```

Alright cool, at this point you've got KubeVirt installed up!

## Setup SR-IOV on-disk configuration file `/etc/pcidp/config.json`

For this step, we're going to use a helper script. I took this from an existing (and open at the time of writing this article) [pull request](https://github.com/kubevirt/kubevirt/pull/1970/files#diff-b5c3b8dc9c9c9b830387bcdc7ba05a75), and I put it into [this gist](https://gist.github.com/dougbtv/1d83c233975e3444957e318f39949d14).

I went ahead and did this as root on each node that has SR-IOV devices (in my case, just one machine)

```
# curl -s https://gist.githubusercontent.com/dougbtv/1d83c233975e3444957e318f39949d14/raw/ef0bcad7e4a318b3791934ff60a87cc40c4233a9/sriov-helper.sh > sriov-helper.sh
# chmod +x sriov-helper.sh
# ./sriov-helper.sh
```

Now we can inspect the contents of the file...

```
# cat /etc/pcidp/config.json
```

On my machine I can see that the `rootDevices` matches what I initialized in my SR-IOV setup way above in this article, specifically `2f:00.2`.

### Restart the SR-IOV device plugin pods...

Now that this is setup, you have to delete the SR-IOV pods... Back to the master (or wherever your kubectl command is run from).

Give this a try...

```
$ kubectl get pods --namespace=sriov | grep device-plugin | awk '{print $1}' | xargs -L 1 -i kubectl delete pod {} --namespace=sriov
```

If it stalls out (full disclosure, mine did), you can just list them and delete one-by-one.

```
$ kubectl get pods --namespace=sriov -o wide | grep device-plugin
```

and then with each one:

```
$ kubectl delete pod $each_pod_name_here --namespace=sriov
```

And then just to make sure, I took the one pod running on my host with SR-IOV devices and looked at the logs...

```
$ kubectl logs kube-sriov-device-plugin-nblww --namespace=sriov
```

In this case, I could see the last line was a `ListAndWatch(sriov)` log and it had content about my device, looked something like this:

```
&ListAndWatchResponse{Devices:[&Device{ID:0000:2f:0a.0,Health:Healthy,}
```

## Let's start a (vanilla) Virtual Machine!

Move back to your master (or wherever your run Kubevirt from), and we're going to spin up a vanilla VM just to get the commands down and make sure everything's looking [hunky dory](https://www.etymonline.com/word/hunky-dory).

First we'll clone the kubevirt repo (word to the wise, it's pretty big, maybe 400 meg clone).

```
$ git clone https://github.com/kubevirt/kubevirt.git --depth 50 && cd kubevirt
```

Let's move into the example VMs section...

```
$ cd cluster/examples/
```

And edit a file in there, let's edit the `vm-cirros.yaml` -- a [classic test VM image](https://launchpad.net/cirros). Bring it up in your editor first, but, we'll edit in place like so:

```
$ sed -ie "s|registry:5000/kubevirt/cirros-container-disk-demo:devel|kubevirt/cirros-container-disk-demo:latest|" vm-cirros.yaml
```

Kubectl create from that file...

```
$ kubectl create -f vm-cirros.yaml
```

And let's look at the `vms` custom resources, and we'll see that it's created, but, not yet running.

```
$ kubectl get vms
NAME        AGE     RUNNING   VOLUME
vm-cirros   2m13s   false     
```

Yep, it's not started yet, let's start it...

```
$ virtctl start vm-cirros
VM vm-cirros was scheduled to start
$ kubectl get vms
NAME        AGE     RUNNING   VOLUME
vm-cirros   3m17s   true      
```

Wait for it to come up (watch the pods...), and then we'll console in (you can see that the password is listed right there in the MOTD, `gocubsgo`). You might have to hit `<enter>` to see the prompt.

```
[centos@kube-nonetwork-master examples]$ virtctl console vm-cirros
Successfully connected to vm-cirros console. The escape sequence is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
vm-cirros login: cirros
Password: 
$ echo "foo"
foo
```

(You can hit `ctrl+]` to get back to your command line, btw.)

## Presenting... a VM with an SR-IOV interface!

Ok, back into your master, and still in the examples directory... Let's create the SR-IOV example. First we change the image location again...

```
sed -ie "s|registry:5000/kubevirt/fedora-cloud-container-disk-demo:devel|kubevirt/fedora-cloud-container-disk-demo:latest|" vmi-sriov.yaml
```

Create a network configuration, a `NetworkAttachmentDefinition` for this one...

```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/sriov
spec: 
  config: '{
    "type": "sriov",
    "name": "sriov-net",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.100.0/24",
      "rangeStart": "192.168.100.171",
      "rangeEnd": "192.168.100.181",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "192.168.100.1"
    }
  }'
EOF
```

(*Side note*: The IPAM section here isn't actually doing a lot for us, in theory you can have `"ipam": {},` instead of this setup with the host-local plugin -- I struggled with that a little bit, so, I included here an otherwise dummy IPAM section)

Console in with:

```
virtctl console vmi-sriov
```

Login as `fedora` (with password `fedora`), become root (`sudo su -`) create an `ifcfg-eth1` script:

```
[root@vmi-sriov2 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE="eth1"
BOOTPROTO="none"
IPADDR=192.168.100.2
NETMASK=255.255.255.0
ONBOOT="yes"
TYPE="Ethernet"
```

And:

```
# ifup eth1
```

You can now check out what the configs look like with: `ip a`.

Now -- repeat this for a second VM. I copied the `vmi-sriov.yaml` to another file and changed the metadata->name to `vmi-sriov2`.

I then also created a `/etc/sysconfig/network-scripts/ifcfg-eth1` and assigned a static IP address of `192.168.100.2`. 

We'll reference that IP address later when we create our VoIP workload.

Once you have those two together -- you can probably make a ping between the two workloads, and... You can put your own workload in!

Or, if you like, you can also create a VoIP workload using Asterisk as I did.

## Asterisk configuration

Install asterisk from RPM, in both VMs, install like so:

```
yum install -y asterisk-pjsip asterisk asterisk-sounds-core-en-ulaw
```

Next, we're going to setup our `/etc/asterisk/pjsip.conf` file on both VMs. This creates a SIP trunk between each machine.

```
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0

[alice]
type=endpoint
transport=transport-udp
context=endpoints
disallow=all
allow=ulaw
aors=alice

[alice]
type=identify
endpoint=alice
match=192.168.100.3/255.255.255.255

[alice]
type=aor
contact=sip:anyuser@192.168.100.3:5060

[bob]
type=endpoint
transport=transport-udp
context=endpoints
disallow=all
allow=ulaw
aors=bob

[bob]
type=identify
endpoint=bob
match=192.168.100.2/255.255.255.255

[bob]
type=aor
contact=sip:anyuser@192.168.100.2:5060
```

Once you've loaded that, console into the VM and issue:

```
# asterisk -rx 'pjsip reload'
```

Next we're going to create a file `/etc/asterisk/extensions.conf` which is our "dialplan" -- this tells Asterisk how to behave when a call comes in our trunk. In our case, we're going to have it answer the call, play a sound file, and then hangup.

Create the file as so:

```
[endpoints]
exten => _X.,1,NoOp()
  same => n,Answer()
  same => n,SayDigits(1)
  same => n,Hangup()
```

Next, you're going to tell asterisk to reload this with:

```
# asterisk -rx 'dialplan reload'
```

Now, from the first VM with the `192.168.100.2` address, go ahead and console into the VM and run `asterisk -rvvv` to get an Asterisk console, and we'll set some debugging output on, and then we'll originate a phone call:

```
vmi-sriov*CLI> pjsip set logger on
vmi-sriov*CLI> rtp set debug on
vmi-sriov*CLI> channel originate PJSIP/333@bob application saydigits 1
```

You should see a ton of output now! You'll see the SIP messages to initiate the phone call, and then you'll see information about the RTP (real-time protocol) packets that include the voice media going between the machines!

Awesome! Thanks for sticking with it, now... For your workload to the rescue!
