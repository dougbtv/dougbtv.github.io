---
author: dougbtv
comments: true
date: 2017-05-30 17:02:03-05:00
layout: post
slug: vnf-asterisk-kubernetes
title: VNFs in Kubernetes? Sure thing, here's vnf-asterisk!
category: nfvpe
---

Want to run a virtual network function (VNF) on Kubernetes? You're in luck! This article comprises a small "do it yourself workshop" that I've put together for a talk that I'm giving at [OPNFV Summit](http://events.linuxfoundation.org/events/opnfv-summit) during the [CNCF day co-located event](http://events.linuxfoundation.org/events/opnfv-summit/extend-the-experience/cncf). Today, we're going to use [vnf-asterisk](https://github.com/redhat-nfvpe/vnf-asterisk) which is an open source demo VNF we've created on the [NFVPE devops squad](https://github.com/redhat-nfvpe) to validate various infrastructure deployments and explore other topics such as container networking, scale, HA, and on and on. I've documented it end-to-end as much as possible so participants can go ahead and dissect it to see how I've componentized it, and as well as how you might start to scale it. The requirements are thick, but are based on previous labs on this blog. Ready for (virtual) dialtone in Kube, let's go!

![vnf asterisk logo](https://github.com/dougbtv/vnf-asterisk-controller/raw/master/docs/vnf-asterisk-controller-logo.png)

I've also submitted a talk, along with Leif Madsen about running this VNF for [Astricon 2017](http://www.asterisk.org/community/astricon-user-conference), so we'll see if it makes it in there too. Edit: Updated to add, we are having a [day-of-learning at Astricon 2017 -- we're having 4 workshops](https://astricon2017.sched.com/event/BiRG/workshop-asterisk-as-a-virtual-network-function-session-1-intro-to-nfv-vnf-asterisk), please come and hang out and give it a try with us in person, we're more than happy to help you out, and there is much hacking to be had!

The main take-away for folks here should be A. some nice exposure to how you might both take apart the pieces to containerize them, and also how to knit them back together with Kubernetes (and some Kubernetes usage), but also B. To use as a reference, and to decide what you do and do not like about it. It's OK to not like some of it! In fact, I hope it helps you form your own opinions. In fact, while I do have some opinions -- I try to keep them loose as these technologies grow and gain maturity. There's already things here that I would change, and certain things that are done as a stop gap. 

So enough blabbering, let's fire up some terminals and get to the good stuff!

## Requirements

TL;DR:

* Kube cluster on CentOS
* Persistent storage
* Git (and your favorite editor) on the master
* Ansible (if you're using my lab playbooks) on "some convenient machine"
* Approximately 5 gigs free for docker images

You're going to need a Kubernetes lab environment that has some persistent storage available. Generally my articles also assume a CentOS environment, and while that may not be entirely applicable here, you should know that's where I start from and might color some of the ancillary tools that I employ.

Additionally, you need git (and probably your favorite text editor) on the master node of your cluster.

But if that seems overwhelming? Don't let it be! I've got you covered with these two labs that will get you up and running. All you really need is a machine to use as a virtual machine host, and Ansible installed.

* First use my [kubernetes on CentOS lab](http://dougbtv.com/nfvpe/2017/02/16/kubernetes-1.5-centos/) which uses my [kube-centos-ansible playbooks](https://github.com/redhat-nfvpe/kube-centos-ansible)
* Then, add to it my lab on [glusterFS for backing persistent volume stores on Kubernetes](http://dougbtv.com//nfvpe/2017/08/10/gluster-kubernetes/)

Naturally, if you have another avenue to achieve the same, then go for it!

## Browsing the components

If you'd like to explore the code behind this (and I highly recommend that you do), there's generally two repositories you want to look at:

* [vnf-asterisk](https://github.com/redhat-nfvpe/vnf-asterisk) proper
* [vnf-asterisk-controller](https://github.com/dougbtv/vnf-asterisk-controller)

The controller is a full-stack-javascript web app that both exposes an API and also talks to Asterisk's ARI (Asterisk RESTful Interface) in order to specify behaviors during call flow, and to use [sorcery](https://wiki.asterisk.org/wiki/display/AST/Sorcery) which we use to dynamically configure Asterisk. This is intended to make for a kind of clay infrastructure so that we can mold to fit a number of scenarios for which we can use Asterisk. A lot of people hear Asterisk and think "Oh, IP-PBX". Sure, you could use it for that. But, that's not all. It could be an [IVR](https://en.wikipedia.org/wiki/Interactive_voice_response) (psst, IVR is NOT just an auto-attendent), maybe you could use it on your session border as a B2BUA to hide topology, maybe you'll make a feature server, maybe you'll front a cluster of all of the above with it, maybe you'll use it as a class-4 switch instead of the assumption of class-5 switching with a PBX. There's a lot you can do with it! Here what we're trying to achieve is a flexible way to use the components herein. 

While you're surfing around in the vnf-asterisk repository, you might notice that there's also other notes and possibly Ansible playbooks. There's also exploration we've done here with starting with a legacy, automating that legacy, and then breaking apart the pieces.

If you're looking for the Dockerfiles for all the pieces, you're going to want to look in a few places, `vnf-asterisk-controller` but also in the [docker-asterisk](https://github.com/dougbtv/docker-asterisk) repo, and also the [homer-docker](https://github.com/sipcapture/homer-docker) repo.

Last but not least -- this also includes [Homer](https://github.com/sipcapture/homer); a VoIP capture, troubleshooting & monitoring tool, which I enjoy contributing too (and using even more!), and I designed the PoC method by which Homer is deployed in Kubernetes, and have maintained the Dockerfiles / docker-compose methodology for a few years. 

Don't deny Homer here, and take it's lesson for your own deployments -- implement monitoring and logging like you mean it. Homer has saved my bacon a number of times, seriously.

## Basic setup.

Generally speaking, we'll do this work from the Kubernetes master server. If you have `kubectl` setup in another place, go ahead and use whatever can access your Kubernetes cluster. 

Now that you have a kubernetes cluster up with persistent volume storage (also, congrats!) you should first check that you can use kube DNS from the master. My lab playbooks don't currently account for this, so that's going to be the first thing we do. It's worth the effort to make the rest of the steps easier without having to poke around too too much. Necessary evil, but, we're onto the fun stuff in a moment.

### DNS

We'll use nslookup, so let's make sure that's around.

```
[centos@kube-master ~]$ sudo yum install -y bind-utils
```

And you should see if you can resolve `kubernetes.default.svc.cluster.local` (the address of the kube api) -- If you can great! Skip the rest of the DNS setup. Otherwise, we'll patch this up in a second.

```
[centos@kube-master ~]$ nslookup kubernetes.default.svc.cluster.local
Server:   192.168.122.1
Address:  192.168.122.1#53

** server can't find kubernetes.default.svc.cluster.local: NXDOMAIN

[centos@kube-master ~]$ echo $?
1
```

Great, as expected, it doesn't work. So we're just going to modify our `/etc/resolv.conf`. So figure out which address is for kube dns.

```
[centos@kube-master ~]$ kubectl get svc --all-namespaces | grep dns | awk '{print $3}'
10.96.0.10
```

Now, in your favorite editor (hopefully not emacs, heaven forbid) go ahead and alter `resolv.conf` to use this `search` domain, and add the above IP as a resolver.

It should now look something like:

```
[centos@kube-master ~]$ cat /etc/resolv.conf 
; generated by /usr/sbin/dhclient-script
search cluster.local
nameserver 10.96.0.10
nameserver 192.168.122.1
```

Note: That won't be sticky through reboots. I'll leave that as an exercise for my readers (or someone can make a PR on my playbooks!)

And just make sure that it works.

```
[centos@kube-master ~]$ nslookup kubernetes.default.svc.cluster.local
Server:   10.96.0.10
Address:  10.96.0.10#53

Non-authoritative answer:
Name: kubernetes.default.svc.cluster.local
Address: 10.96.0.1

[centos@kube-master ~]$ echo $?
0
```

## Let's run vnf-asterisk!

You're not too far away, just got to clone the repo.  let's go ahead and run it now. Note that the [official repo is here on Github](https://github.com/redhat-nfvpe/vnf-asterisk), we're not cloning that as we'll use the branch that this tutorial is based on in my fork -- so it keeps working as the official repo changes.

```
[centos@kube-master ~]$ git clone https://github.com/dougbtv/vnf-asterisk.git
[centos@kube-master ~]$ cd vnf-asterisk/
[centos@kube-master vnf-asterisk]$ git checkout containers
[centos@kube-master vnf-asterisk]$ cd k8s/ansible/roles/load-podspec/templates/
[centos@kube-master templates]$ ls
homer-podspec.yml.j2  podspec.yml.j2
```

You'll see two resource definition files in there. They're `.j2` jinja2 files, but, ignore that, there's no templating in them now. You could also run the ansible playbooks to template these onto the machine (really handy for development of the vnf-asterisk application), but, it's enough steps to make it easier to just clone this.

We're going to go ahead and create everything given those, so run yourself `kubectl` with those two files.

```
[centos@kube-master templates]$ kubectl create -f podspec.yml.j2 -f homer-podspec.yml.j2 
```

Watch everything while it comes up -- it's going to pull A LOT OF IMAGE FILES. Around 4 gigs. Yeah, that's less than idea. Some of these just had to be bigger, maybe I can improve that later. It's a pain when pulling from a public registry, but, in a local registry -- it's not so terribly bad. That could make it take quite a while.

I watch it come up with:

```
[centos@kube-master templates]$ watch -n1 kubectl get pods
```

### Trouble shooting that deploy

If everything is coming up in `kubectl get pods` with a status of `Running` -- you're good to go!

If you've been toying around with the persistent volumes from my lab, and you see pods failing you might need to recreate them, I had to do `kubectl delete -f ~/glusterfs-volumes.yaml` and then `kubectl create -f ~/glusterfs-volumes.yaml`. 

Also generally, double check -- is something still pulling for an image? It could be, and it takes a long time. So, double check that.

Otherwise, you can do the usual where you do `kubectl get pods` and for a particular pod that's not in a running state, do a `kubectl describe pod somepod-1550262015-x6v8b`.

## Checking out the running pieces.

So, what is running? Let's look at my `get pods` output.

```
[centos@kube-master templates]$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
asterisk-2725520970-w5mnj   2/2       Running   0          8m
controller                  1/1       Running   0          8m
cron-1550262015-x6v8b       1/1       Running   0          8m
etcd0                       1/1       Running   0          8m
etcd1                       1/1       Running   0          8m
etcd2                       1/1       Running   0          8m
kamailio-2669626650-tg855   1/1       Running   2          8m
mysql-1479608569-4tx26      1/1       Running   0          8m
vnfui                       1/1       Running   0          8m
webapp-3687268953-ml1t4     1/1       Running   0          8m
```

You'll see there's some interesting things:

* An asterisk instance (one of them)
* A controller (a REST-ish API that can control asterisk)
* A vaguely named "webapp" -- which is homer's web UI
* A "vnfui" which is the web UI for the vnf-asterisk-controller
* etcd -- a distributed key/value used for service discovery
* cron - Cron jobs for Homer (to later become Kubernetes cron-type jobs)
* MySQL - used for Homer's storage
* kamailio - a SIP proxy, here used by Homer to look at VoIP traffice

If you do `kubectl get pods --show-all` you'll also see a bootstrap job which prepopulates the data structures used by Homer.

You'll also note the "asterisk" pod is the lone pod with `2/2` ready -- as it has two containers. It has both asterisk proper, and a [captagent](https://github.com/sipcapture/captagent) to capture VoIP traffic, which it sniffs out of the shared network interface in the infra-container for the pod which both containers share.

And the services available:

```
[centos@kube-master templates]$ kubectl get svc
NAME                CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
bootstrap           None             <none>        55555/TCP           9m
controller          None             <none>        8001/TCP            9m
cron                None             <none>        55555/TCP           9m
etcd-client         10.104.94.90     <none>        2379/TCP            9m
etcd0               10.107.105.188   <none>        2379/TCP,2380/TCP   9m
etcd1               10.111.0.145     <none>        2379/TCP,2380/TCP   9m
etcd2               10.101.9.115     <none>        2379/TCP,2380/TCP   9m
glusterfs-cluster   10.99.161.63     <none>        1/TCP               4d
kamailio            10.111.59.35     <none>        9060/UDP            9m
kubernetes          10.96.0.1        <none>        443/TCP             5d
mysql               None             <none>        3306/TCP            9m
vnfui               None             <none>        80/TCP              9m
webapp              10.102.94.75     <none>        8080/TCP            9m
```

At the command line we can validate that a few things are running.

First, we have a controller running, it's an API that can control what our Asterisk machines are doing. Just bring up the `/foo` endpoint to see that it's working at all.

```
[centos@kube-master templates]$ curl controller.default.svc.cluster.local:8001/foo && echo
[{"text":"this and that"},{"text":"the other thing"},{"text":"final"}]
```

Now, if that's working well, that's a good sign.

Here's running an Asterisk command, we can see we have one instance of Asterisk.

```
[centos@kube-master templates]$ kubectl exec -it $(kubectl get pods | grep asterisk | tail -n1 | awk '{print $1}') -- asterisk -rx 'core show version'
Asterisk 14.3.0 built by root @ 1b0d6163fdc2 on a x86_64 running Linux on 2017-03-01 20:49:29 UTC
```

You can also bring up an interactive prompt too if you wish.

```
[centos@kube-master templates]$ kubectl exec -it $(kubectl get pods | grep asterisk | tail -n1 | awk '{print $1}') -- asterisk -rvvv
[... snip ...]
asterisk-2725520970-w5mnj*CLI> 
```

## Bring it up in browser! Create the tunnels for the lab machines in VMs (Optional)

If your lab is like mine (e.g. You've used my lab playbooks to create VMs on a virt host to run a Kubernetes cluster), the VMs running Kubernetes are walled off inside their own network. So you'll have to create some tunnels in. This is... Less than convenient. Given this is a lab, it doesn't have great network facilities for ingress. So, it's fairly manual, sorry about that. Personally I'm frustrated with this, so my apologies are sincere. Maybe another blog article coming in the future for making the networking scenario a bit more user-friendly to access these services from afar.

Ok, first on the master let's collect the IP addresses that we'll need to forward. This bash command is a mouthful, but, it'll give us the IPs we need, and we'll use those on.

```
[centos@kube-master templates]$ podstring="controller vnfui webapp"; \
  for pod in $podstring; do \
    ip=$(kubectl get svc | grep $pod | awk '{print $2}'); \
    echo $pod=$ip; \
  done
controller=10.244.3.17
vnfui=10.244.1.16
webapp=10.244.1.18
```

Now that you have that, let's paste those as variables into our workstation.

```
[doug@workstation ~]$ controller=10.244.3.17
[doug@workstation ~]$ vnfui=10.244.1.16
[doug@workstation ~]$ webapp=10.244.1.18

```

And dig up the IP addresses for both the virtual machine host and your master, and we'll set those as variables too. Again if you're using my lab playbooks those are in your `vms.inventory`

Let's set those as variables now, too. In my case, my virt host is ``

```
[doug@workstation ~]$ jumphost=192.168.1.119
[doug@workstation ~]$ masterhost=192.168.122.151
```

Now you can setup all the jumphost tunneling like so:

```
[doug@workstation ~]$ ssh -L 8088:localhost:8088 -L 8001:localhost:8001 -L 8080:localhost:8080 -t root@$jumphost ssh -L 8088:$nginx:80 -L 8001:$controller:8001 -L 8080:$webapp:8080 -t -i .ssh/id_vm_rsa centos@$masterhost
```

And from your workstation, you should be able to test out the controller:

```
$ curl localhost:8001/foo && echo
```

You can access the web UI for the controller is @ `http://localhost:8088`

The web UI for Homer (VoIP analytics tool) is @ `http://localhost:8080`

## Scale it UP!

So what we're about to do now is take this default setup we have. And scale up a little bit. Once we scale up, we'll provision SIP trunks between the Asterisk instances, and then we'll make a call over it, and check out the analytics that we have setup.

You'll note that we're doing a bunch manually here. This could all theoretically be automated, including the API calls we'll make to the customized controller I created. But, in the name of educating you about how it all works, we're going to do this manually for now.

### Scale up Asterisk instances

First thing we can do here is check out the [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment) that was specified in our yaml resource definitions. 

```
[centos@kube-master ~]$ kubectl describe deployment asterisk | grep -P "^Replicas"
Replicas:       1 desired | 1 updated | 1 total | 1 available | 0 unavailable
```

This shows us that our deployment requested a single instance, and 1 is up. So let's scale that up to two instances.

```
[centos@kube-master ~]$ kubectl scale deployment asterisk --replicas 2
deployment "asterisk" scaled
```

Now check out our `kubectl describe` again.

```
[centos@kube-master ~]$ kubectl describe deployment asterisk | grep -P "^Replicas"
Replicas:       2 desired | 2 updated | 2 total | 2 available | 0 unavailable
```

And we'll see that there's two pods available.

```
[centos@kube-master ~]$ kubectl get pods | grep -P "(NAME|asterisk)"
NAME                        READY     STATUS    RESTARTS   AGE
asterisk-2725520970-dwb93   1/1       Running   0          59s
asterisk-2725520970-tz31p   1/1       Running   0          1h
```

That's good news, we've got two instances. 

### Provision trunks

Now that we have our two instances, we can create trunks over them. Let's use the master and we'll use the [vnf-asterisk-controller](https://github.com/dougbtv/vnf-asterisk-controller) to help us do this. If you're curious about what the vnf-asterisk-controller can do, check that out. There's also a [API blueprint on Apiary.io](http://docs.vnfasteriskcontroller.apiary.io/) describing all the API functionality if you're interested.

These instances have a entrypoint script which announces their presence to etcd for service discovery, and the controller can discover these endpoints. Once the endpoints are discovered, we can then instruct the controller to create a SIP trunk between the two.

So, let's go ahead and call the controller's `/discover` endpoint. 

```
[centos@kube-master ~]$ [centos@kube-master ~]$ curl -s controller.default.svc.cluster.local:8001/discover | python -m json.tool
[
    {
        "ip": "10.244.3.20",
        "nickname": "suspicious_shaw",
        "trunks": [],
        "uuid": "f7feaa73-e823-4d47-b4f4-3310aa548bcb"
    },
    {
        "ip": "10.244.1.23",
        "nickname": "lonely_meitner",
        "trunks": [],
        "uuid": "b0e7990a-7009-4b00-9614-e0973da8ee68"
    }
]
```

You can see that there's two Asterisk machines discovered by the controller, using etcd.

Additionally -- if you bring up the vnfui, you can see these endpoints there in the Web UI.

Here's what it looks like in the web UI:

![vnf asterisk web ui](http://i.imgur.com/3DDN3x2.png)

You'll note there's two `nickname` items there. This is just a shortcut that I built in that allows us to call them something other than the `uuid` for fun. I used a script (as a service) inspired by the Docker container naming scheme there to do this. These nicknames are random, so, yours will (almost certainly) differ.

But, we're going to use the UUIDs for now. Here's how you can pick up those UUIDs

```
[centos@kube-master ~]$ uuida=$(curl -s controller.default.svc.cluster.local:8001/discover | python -m json.tool | grep uuid | awk '{print $2}' | sed -s 's/[^a-z0-9\-]//g' | tail -n1)
[centos@kube-master ~]$ uuidb=$(curl -s controller.default.svc.cluster.local:8001/discover | python -m json.tool | grep uuid | awk '{print $2}' | sed -s 's/[^a-z0-9\-]//g' | head -n1)

[centos@kube-master ~]$ echo $uuida
b0e7990a-7009-4b00-9614-e0973da8ee68
[centos@kube-master ~]$ echo $uuidb
f7feaa73-e823-4d47-b4f4-3310aa548bcb
```

Now that we have those, we can use the `connect` API endpoint of the controller.

```
[centos@kube-master ~]$ curl -s controller.default.svc.cluster.local:8001/connect/$uuida/$uuidb/inbound 
```

You'll get some JSON back about the trunks created. But, we can also pick that up from the discover endpoint, it should look like:

```
[centos@kube-master ~]$ curl -s controller.default.svc.cluster.local:8001/discover | python -m json.tool
[
    {
        "ip": "10.244.3.20",
        "nickname": "suspicious_shaw",
        "trunks": [
            "/asterisk/f7feaa73-e823-4d47-b4f4-3310aa548bcb/trunks/lonely_meitner"
        ],
        "uuid": "f7feaa73-e823-4d47-b4f4-3310aa548bcb"
    },
    {
        "ip": "10.244.1.23",
        "nickname": "lonely_meitner",
        "trunks": [
            "/asterisk/b0e7990a-7009-4b00-9614-e0973da8ee68/trunks/suspicious_shaw"
        ],
        "uuid": "b0e7990a-7009-4b00-9614-e0973da8ee68"
    }
]
```

You'll see in the `trunks` list there's a path to the trunks, and it will have the nickname of the partner at the other end of the SIP trunk.

### Inspecting the results in Asterisk.

So -- which is which? This is part of the reason that we have these nicknames. Let's figure out who's who. Let's pull up the pod name for the first instance -- we're going to fish it out of the logs. 

```
[centos@kube-master ~]$ kubectl logs $(kubectl get pods | grep asterisk | head -n1 | awk '{print $1}') -c asterisk | grep "Announcing nick"
Announcing nickname to etcd: lonely_meitner
+ echo 'Announcing nickname to etcd: lonely_meitner'
```

So we can see the first instance is `lonely_meitner`. Cool.

Now, with that in hand, let's also check out the trunks that have been built in Asterisk.

```
[centos@kube-master ~]$ kubectl exec -it $(kubectl get pods | grep asterisk | head -n1 | awk '{print $1}') -c asterisk -- asterisk -rx 'pjsip show endpoints' 

 Endpoint:  <Endpoint/CID.....................................>  <State.....>  <Channels.>
    I/OAuth:  <AuthId/UserName...........................................................>
        Aor:  <Aor............................................>  <MaxContact>
      Contact:  <Aor/ContactUri..........................> <Hash....> <Status> <RTT(ms)..>
  Transport:  <TransportId........>  <Type>  <cos>  <tos>  <BindAddress..................>
   Identify:  <Identify/Endpoint.........................................................>
        Match:  <ip/cidr.........................>
    Channel:  <ChannelId......................................>  <State.....>  <Time.....>
        Exten: <DialedExten...........>  CLCID: <ConnectedLineCID.......>
==========================================================================================

 Endpoint:  suspicious_shaw                                      Not in use    0 of inf
        Aor:  suspicious_shaw                                    0
      Contact:  suspicious_shaw/sip:anyuser@10.244.3.20:50 05ea73df04 Unknown         nan
  Transport:  transport-udp             udp      0      0  0.0.0.0:5060
   Identify:  suspicious_shaw/suspicious_shaw
        Match: 10.244.3.20/32
```

Cool, we can see that `lonely_meitner` is connected to `suspicious_shaw`. You might also want to check out `pjsip show aors`.

### Make a call

Now, let's make a call between these boxes. Instead of trying to guess which one comes up first in yours, I'm going to let you copy and paste your own trunk name, and insert it into the command here. So substitute the nickname of the other host here in this command.

In fact, mine were backwards by the time I tried this. So go ahead and execute the asterisk command line one of them, and then do `pjsip show aors` and show the trunk name.

```
[centos@kube-master ~]$ kubectl exec -it $(kubectl get pods | grep asterisk | tail -n1 | awk '{print $1}') -- asterisk -rvvv

asterisk-2725520970-tz31p*CLI> pjsip show aors

      Aor:  <Aor..............................................>  <MaxContact>
    Contact:  <Aor/ContactUri............................> <Hash....> <Status> <RTT(ms)..>
==========================================================================================

      Aor:  lonely_meitner                                       0
    Contact:  lonely_meitner/sip:anyuser@10.244.1.23:5060  ee623310fc Unknown         nan
```

Now go ahead and originate the call substituting your trunk name for `lonely_meitner`

```
asterisk-2725520970-tz31p*CLI> channel originate PJSIP/333@lonely_meitner application wait 5
    -- Called 333@lonely_meitner

    -- PJSIP/lonely_meitner-00000004 answered
```

Now, we've had a call happen! Let's go ahead and checkout some call detail records (CDRs).

```
[centos@kube-master ~]$ kubectl exec -it $(kubectl get pods | grep asterisk | tail -n1 | awk '{print $1}') -c asterisk -- cat /var/log/asterisk/./cdr-csv/Master.csv
"","anonymous","333","inbound","""Anonymous"" <anonymous>","PJSIP/desperate_poitras-00000000","","Hangup","","2017-05-31 19:49:18","2017-05-31 19:49:18","2017-05-31 19:49:18",0,0,"ANSWERED","DOCUMENTATION","1496260158.0",""
```

And you can see that it logged the call! Hurray.

### Check it out in Homer

Now, let's check out what's going on with Homer. Homer is a VoIP analytics and monitoring tool. It can show you what's up with your SIP traffic for one. And does some pretty sweet stuff like.

First, let's peek at the database. You'll note here I'm figuring out the name of MySQL pod, then I'm going to exec a MySQL CLI from that pod. You'll note everything is insecure about this MySQL instance and how it's called. Like, it's using root and the password is "secret" and I use the password on the command-line. "Do as I say, not as I do" as it has been said, yeah... Just don't do any of that stuff, this is a demo after all. 

```
[centos@kube-master ~]$ kubectl get pods | grep mysql | awk '{print $1}'
mysql-1479608569-7bgnw
[centos@kube-master ~]$ kubectl exec -it mysql-1479608569-7bgnw -- mysql -u root -p'secret'

[... snip ...]

mysql> # What day is today? We'll use this to get our table name
mysql> SELECT DATE(NOW());                                      
+-------------+
| DATE(NOW()) |
+-------------+
| 2017-05-31  |
+-------------+
1 row in set (0.00 sec)

mysql> # Now, use that date in the table name and select from it.
mysql> SELECT id,`date`,method,ruri,ruri_user,user_agent FROM homer_data.sip_capture_call_20170531 LIMIT 1\G
*************************** 1. row ***************************
        id: 1
      date: 2017-05-31 19:49:18
    method: INVITE
      ruri: sip:333@10.244.3.24:5060
 ruri_user: 333
user_agent: Asterisk PBX 14.3.0
1 row in set (0.00 sec)

```

There it is!

And if you are able to, we can also bring that up in the UI.

If you have my lab setup, bring up `http://localhost:8080` and then use username "admin" password "test123". 

Click the "clock icon" in the upper right hand corner, and select say "Last 24 hours" then hit the search button (window pane under the nav towards the left).

Now if you hit the call ID in the results there, it should bring up a "ladder diagram" (which I recall from even the ISDN days! But, is standard for the SIP protocol).

Here's what mine looks like:

![homer ladder diagram](http://i.imgur.com/QtdODHA.png)

## In review...

Hurray! And there.... You have it.

The bottom line is -- a lot of what's here for configuring the service once it's up, especially with regards to interacting with the controller & scaling is rather manual; which is to demonstrate how this approach works and let you touch some parts.

You could however, automate those portions, and use some of Kubernetes autoscaling features to make this a lot more automatic & dynamic. Something to think about as you try this out, or as you design your own.

