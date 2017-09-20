---
author: dougbtv
comments: true
date: 2017-09-20 15:00:00-05:00
layout: post
slug: crio-workflow
title: Ghost Riding The Whip -- A complete Kubernetes workflow without Docker, using CRI-O, Buildah & kpod
category: nfvpe
---

It is my decree that whenever you are using Kubernetes without using Docker you are officially "[ghost riding the whip](https://en.wikipedia.org/wiki/Ghost_riding)", maybe even "ghost riding the kube". (Well, I'm from Vermont, so I'm more like "[ghost riding the combine](https://www.youtube.com/watch?v=ak3Dbh2cukc)"). And again, we're running Kubernetes without Docker, but this time? We've got an entire workflow without Docker. From image build, to running container, to inspecting the running containers. Thanks to the good folks from [the OCI project](https://www.opencontainers.org/) and [Project Atomic](https://www.projectatomic.io/), we've got [kpod](https://medium.com/cri-o/announcing-the-first-cri-o-release-candidate-5a57667bc39d) for working with running containers, and we've got [buildah](https://github.com/projectatomic/buildah) for building our images. And of course, don't leave out [CRI-O](http://cri-o.io/) which makes the magic happen to get it all running in Kube without Docker. Fire up your terminals, because you're about to [ghost ride the kube](https://youtu.be/BABrSDGIBl8?t=2m2s).

I happened to see that [there is a first release candidate of CRI-O](https://medium.com/cri-o/announcing-the-first-cri-o-release-candidate-5a57667bc39d) which has a bunch of great improvements that work towards really getting CRI-O production ready for Kubernetes. And I have to say -- my experience with using it has been nearly flawless. It's been working like a champ, and I can tell they're doing an excellent job with the polish. Of course that's awesome, but, I was most excited to hear about `kpod` -- "the missing tool". When I wrote my [first article about using CRI-O](http://dougbtv.com/nfvpe/2017/06/29/kubernetes-crio/), I was missing a few portions -- especially a half decent tool for checking out what's going on with containers. This tool isn't quite as mature as CRI-O itself, but, the presence of this tool at all is just a straight-up boon.

To get this all going, I have these tools (CRI-O, kpod & buildah) integrated into my vanilla kubernetes lab playbooks, [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible). This playbook has it so we can compile CRI-O (which includes kpod), buildah, and get Kubernetes up and running (which uses kubeadm to initialize and join the pods). I made some upgrades to kube-centos-ansible in the process, fixing up [issues with kube 1.7](https://github.com/redhat-nfvpe/kube-centos-ansible/issues/37), and also improving it so that kube-centos-ansible can also use Fedora. CRI-O itself works wondefully with CentOS, but Buildah needs some kernel functionality that just isn't available in CentOS yet, so... kube-**centos**-ansible now also supports Fedora, oddly or not-so-oddly enough.

## Requirements

This walk-through assumes that you have at least 2 machines with Fedora installed (and generally up-to-date). That's where we'll install Kubernetes with CRI-O (and kpod!). You might notice that we use [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible), the name of which is... Not so apropos. But! It's recently been updated to support Fedora. And we need Fedora to get a spankin' fresh kernel, so we can use... Drum roll please... [Buildah](https://github.com/projectatomic/buildah) -- an image building tool that is not Docker (wink, wink!).

Those machines need to have over 2 gigs of RAM. Compilation of CRI-O, specifically during a step with GCC was bombing out on me with GCC complaining it couldn't allocate memory when I had just 2 gigs of RAM. Therefore, I recommend at least 4 gigs of RAM.

In addition to that, you'll need git & Ansible installed on "some machine" (likely your workstation). And your handy-dandy editor. Cause... How do you live without an editor? Unless you're feeding the input in on punch cards, in which case... You have my respect. 

TL;DR, you need:

* 2 or more Fedora machines with 4 gigs or RAM or more (and maybe 5 gigs free on disk)
* On a client machine (like your workstation)
    - git
    - Ansible
    - An editor (or [punch card reader](http://codeincluded.blogspot.co.nz/2012/07/punchcard-reader-software.html))

## Spinning up a Kubernetes cluster with CRI-O (and kpod included!)

First off, go ahead and clone up the [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible) project...

```
git clone --branch v0.1.3 https://github.com/redhat-nfvpe/kube-centos-ansible.git
```

This article glosses over the fact that the [kube-centos-ansible](https://github.com/redhat-nfvpe/kube-centos-ansible) has the ability to spin-up virtual machines to mock-up Kubernetes clusters. However, if you're familiar with it, you can use it as well. I won't go into depth here, but this is the technique that I use:

```
$ ansible-playbook -i inventory/your.inventory -e "vm_parameters_ram_mb=4096" virt-host-setup.yml 
```

Now we'll a playbook to bootstrap the nodes with Python (as the [Fedora cloud images don't come packaged with Python](https://trello.com/c/XaiXEocS/239-bz-to-track-adding-python-to-the-fedora-cloud-images)). 

```
$ ansible-playbook -i inventory/your.inventory fedora-python-bootstrapper.yml
```

For your reference here's the inventory I used. This inventory can also be found in the `./inventory/examples/crio/crio.inventory` in the clone. Mostly this is here to show you how to set the variables in order to get this puppy (that is, kube-centos-ansible) to properly use Fedora, when it comes down to it.

```
kube-master ansible_host=192.168.1.149
kube-node-1 ansible_host=192.168.1.87
kubehost ansible_host=192.168.1.119 ansible_ssh_user=root

[kubehost]
kubehost
[kubehost:vars]
# Using Fedora
centos_genericcloud_url=https://download.fedoraproject.org/pub/fedora/linux/releases/26/CloudImages/x86_64/images/Fedora-Cloud-Base-26-1.5.x86_64.qcow2
image_destination_name=Fedora-Cloud-Base-26-1.5.x86_64.qcow2
increase_root_size_gigs=10

[master]
kube-master
[nodes]
kube-node-1
[all_vms]
kube-master
kube-node-1

[all_vms:vars]
# Using Fedora
ansible_ssh_user=fedora
ansible_ssh_private_key_file=/home/doug/.ssh/id_testvms
kubectl_home=/home/fedora
kubectl_user=fedora
kubectl_group=fedora
```

## Start the Kubernetes install

Then you can go ahead and get your kube install rolling!

```
$ ansible-playbook -i inventory/vms.inventory -e 'container_runtime=crio' kube-install.yml 
```

That, my good friend... Is is a coffee-worthy step. It is now time to fuel up while that runs. (We're compiling some big-idea kind of stuff, like CRI-O [and more]).

## Verify that things are hunky dory

(Still unsure what the genesis of the phrase "hunky dory" is. But it means "satisfactory" or "just fine")

Log yourself into the master. And, first of all things... Make sure you DON'T have Docker. And grin during this step. Cause I sure did.

```
[fedora@kube-crio-master ~]$ docker
-bash: docker: command not found
[fedora@kube-master ~]$ echo $?
127
```

YES. We want that to `exit 127`!

Make sure to see that the nodes are healthy...

```
$ kubectl get nodes
```

And making sure the nodes are in a ready state.

Optionally, spin up a pod. in my case I did a...

```
[fedora@kube-crio-master ~]$ cat <<EOF | kubectl create -f -
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
EOF
[fedora@kube-crio-master ~]$ watch -n1 kubectl get pods
```

They should come up! And if they are you should be able to query nginx.

```
[fedora@kube-crio-master ~]$ curl -s $(kubectl describe pod $(kubectl get pods | grep nginx | head -n 1 | awk '{print $1}') | grep "^IP" | awk '{print $2}') | grep -i thank
<p><em>Thank you for using nginx.</em></p>
```

Cool! That means that you have CRI-O up and poppin'. You are officially ghost riding the whip.

Clean that up if you want, with:

```
[fedora@kube-master ~]$ kubectl delete rc nginx
```

Wait -- didn't I promise you a complete work-flow that omits Docker at all? That's right I did. So let's go ahead and start up a from-scratch workflow here... with...

## Buildah!

Awesome. Now, let's go ahead and log into the node. For ease, for now, we'll also `sudo su -`. In the future, you might wanna set this up to work for a specific user, but, I'll leave that as a journey for the reader.

Check out the help for buildah, if you wish. That's how I learned how to do this myself.

```
[root@kube-node-1 ~]# buildah --help
```

Now, let's create a "Dockerfile". We'll use the Dockerfile syntax, as I'm familiar with it, and if you have existing Dockerfiles -- buildah supports that! 

So go ahead and make yourself a Dockerfile like so.

```
[root@kube-node-1 ~]# cat Dockerfile 
FROM fedora:26
RUN dnf install -y cowsay-beefymiracle cowsay
ENTRYPOINT ["cowsay","-s","Shoutout from Vermont!"]
```

This image is just a couple RPMs, really. Mostly cowsay (and then an extra "cowsay file" to add [the beefy miracle](https://opensource.com/life/12/5/whats-beefy-miracle-anyway-story-fedora-17-release-name) art. According [to Wikipedia](https://en.wikipedia.org/wiki/Cowsay): 

> cowsay is a program that generates ASCII pictures of a cow with a message.

And you think that machine learning is high tech? Obviously you haven't seen cow ASCII art insult a co-worker before. The pinnacle of technology.

**BONUS**: To insult your co-workers using cowsay, install the package with `dnf install cowsay` and use `wall` to broadcast a message to all terminals logged into a machine.

```
[fedora@kube-node-1 ~]$ cowsay -s "your mother wears army boots" | wall
                                                                               
Broadcast message from fedora@kube-node-1 (pts/0) (Wed Sep 20 13:32:41 2017):
                                                                               
 ______________________________                                                
< your mother wears army boots >                                               
 ------------------------------                                                
        \   ^__^                                                               
         \  (**)\_______                                                       
            (__)\       )\/\                                                   
             U  ||----w |                                                      
                ||     ||                                                      
```

Now that you have sufficiently made enemies with your co-workers, back to getting this workflow going.

Go ahead and kick off the build. And on the subject of ASCII -- enjoy yourself the nicer ASCII progress bars than Docker, too.

```
[root@kube-node-1 ~]# buildah  --storage-driver overlay2 bud -t dougbtv/beefy .
```

The command we're using there is `buildah bud` -- `bud` is "build using dockerfile". Very nice feature. 

Note that we're setting `--storage-driver overlay2` (as a global option) which will store the images in the proper locations for `runc` (and therefore CRI-O) to see where these images are.

Also, for what it's worth -- I didn't have great luck with the build cache on subsequent runs of buildah. I'm unsure what the progress on that functionality in buildah itself is. Likely, it may be something I did wrong in the compilation or installation of buildah, so if you see it and shoot me a note on twitter or place a github issue, that'd be awesome.

You can go ahead and list what you just built. Note that we're including the storage driver option, again.

```
[root@kube-node-1 ~]# buildah  --storage-driver overlay2 images | grep -P "(IMAGE|beefy)"
IMAGE ID             IMAGE NAME                                               CREATED AT             SIZE
95c3725439f6         docker.io/dougbtv/beefy:latest                           Sep 15, 2017 23:28     1.983 KB
```

Great! You've got an image. 

## Now, lets run that image!

We'll do this with Kubernetes itself today. Log into your master, and first thing, let's specify a label that we'll use for a node selector (which will specify on which node we'll run this particular pod). In this case we're doing this because we don't have a registry to pull the images from, so, we've got to tell Kube to run the pod in a particular place -- because that where we built the image. 

Here's the (admittedly zany) label that I added. (You can make a lot less insane [node selector constraint](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector) if you're sound of mind, too.)

```
$ kubectl label nodes kube-node-1 beefylevel=expert
```

And you can see what's been labeled with:

```
[fedora@kube-master ~]$ kubectl get nodes --show-labels
```


Create yourself a `beefy.yaml`. Here you'll see a few things that are fairly important, but something to pay attention to in this context is the `imagePullPolicy: Never`. Since this isn't available on a registry, we want to tell Kubernetes "don't even try to pull this", by default it will try, say it can't pull it, and then it won't run the container.

Here's the `beefy.yaml` I created.

```
[fedora@kube-master ~]$ cat beefy.yaml 
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: beefy
  name: beefy
spec:
  containers:
   - command:
       - "/bin/bash"
       - "-c"
       - "cowsay -f /usr/share/cowsay/beefymiracle.cow -s 'shouts from Vermont' && sleep 2000000"
     image: dougbtv/beefy
     name: beefy
     imagePullPolicy: Never
  nodeSelector:
    beefylevel: expert
```

Go ahead and create that...

```
[fedora@kube-master ~]$ kubectl create -f beefy.yaml 
pod "beefy" created
```

And watch it come up.

```
[fedora@kube-master ~]$ watch -n1 kubectl get pods -o wide
```

(Note that it should be saying it's coming up on `kube-node-1`)

Now for the pay day... Let's see it rollin'.

```
[fedora@kube-master ~]$ kubectl logs beefy
 _____________________
< shouts from Vermont >
 ---------------------
              \
                      .---. __
           ,         /     \   \    ||||
          \\\\      |O___O |    | \\||||
          \   //    | \_/  |    |  \   /
           '--/----/|     /     |   |-'
                  // //  /     -----'
                 //  \\ /      /
                //  // /      /
               //  \\ /      /
              //  // /      /
             /|   ' /      /
             //\___/      /
            //   ||\     /
            \\_  || '---'
            /' /  \\_.-
           /  /    --| |
           '-'      |  |
                     '-'
```

Huzzah! We've got Beefy. It's a gosh darned miracle. Dang heckin' good job.

Awesome! That's a whole workflow without Docker. Aww yisss. Now, let's put a cherry on top...

## Let's try out kpod!

Enter kpod! That's the missing tool from my last CRI-O article. We only had some really rudimentary stuff in `runc` that could do this for us. But, the Atomic guys are really tearing it up, and now we've got `kpod` which can do a whole lot more for us.

Now that we have a running container -- We can check it out with kpod. There's a lot more features on the way for kpod, but, for now it gives a nice way to work with your containers (and some container image utilities). I wanted to run it directly with this, but, that's in the works at the tag at which I have CRI-O/kpod pinned.

So go ahead and log into the node... And we'll `sudo su -` for now (as above). And let's list the container processes...

```
[root@kube-node-1 ~]# kpod ps
```

Awesome! You should see `dougbtv/beefy:latest` in there.

And you can list the images with this tool, too.

```
[root@kube-node-1 ~]# kpod images
```

Say you want to see what's in the ephemeral storage of an image, we can use kpod for this, too. So let's pick up the id of our running container.

```
[root@kube-node-1 ~]# beefyid=$(kpod ps | grep -i beef | awk '{print $1}')
[root@kube-node-1 ~]# echo $beefyid
74caa091b27b7
```

Now we can use that in order to look at what's in the container. Let's just cat the definition for the beefy miracle in cowsay.

```
[root@kube-node-1 ~]# cat $(kpod mount $beefyid)/usr/share/cowsay/beefymiracle.cow
```

That should show you a heavily escaped ASCII hotdog. Alright! Nice work Project Atomic folks! Quite a feat.

