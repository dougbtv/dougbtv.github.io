---
author: dougbtv
comments: true
date: 2017-04-20 16:20:01-05:00
layout: post
slug: homer-kubernetes
title: Let's run Homer on Kubernetes!
category: nfvpe
---

I have to say that [Homer](http://sipcapture.org/) is a favorite of mine. Homer is VoIP analysis & monitoring -- on steroids. Not only has it saved my keister a number of times when troubleshooting VoIP platforms, but it has an awesome (and helpful) open source community. In my opinion -- it should be an integral part of your devops plan if you're deploying VoIP apps (really important to have visibility of your... Ops!). [Leif](https://github.com/leifmadsen) and I are using Homer as part of our (still WIP) [vnf-asterisk](https://github.com/redhat-nfvpe/vnf-asterisk) demo VNF (virtualized network function). We want to get it all running in OpenShift & Kubernetes. Our goal for this walk-through is to get Homer up and running on Kubernetes, and generate some traffic using [HEPgen.js](https://github.com/sipcapture/hepgen.js?files=1), and then view it on the Homer Web UI. So -- why postpone joy? Let's use [homer-docker](https://github.com/sipcapture/homer-docker) to go ahead and get Homer up and running on Kubernetes. 

Do you just want to play? You can skip down to the "requirements" section and put some hands on the keyboard.

## Some background

First off, I really enjoy working upstream with the [sipcapture](https://github.com/sipcapture) crew -- they're really nice, and have created quite a fine community around the world of software that comprises Homer. They're really a friendly bunch, and they're always looking to make Homer better -- I'm a regular contributor, and you'll see I proudly contribute as evidenced by my badge showing [membership of the sipcapture org on my github profile](https://github.com/dougbtv)!

This hasn't yet landed in the official upstream homer-docker repo yet. It will eventually, maybe even by the time you're reading this. So, look for a `./k8s` directory in the official [homer-docker](https://github.com/sipcapture/homer-docker) repository. There's a couple things I need to change in order to get it in there -- in part I need to get a build pipeline to get the images into a registry. Because, a registry is required, and frankly it's easy to "just use dockerhub". If you've got a registry -- use your own! You can trust it. Later on in the article, I'll encourage you to use your own registry or dockerhub images if you please, but, also give you the option of using the images I have already built -- so you can just get it to work.

I also need to document it -- that's partially why I'm writing this article, cause I can generate some good docs for the repo proper! And there's at least a few rough edges to sand off (secret management, usage of cron jobs).

That being said, currently -- I have the pieces for this in a fork of the homer-docker repo, in the ['k8s' branch on dougbtv/homer-docker](https://github.com/dougbtv/homer-docker/tree/k8s)

Eventually -- I'd like to make these compatible with OpenShift -- which isn't a long stretch. I'm a fan of running OpenShift; it encourages a lot of good practices, and I think as an organization it can help lower your [bus number](https://en.wikipedia.org/wiki/Bus_factor). It's also easier to manage and maintain, but... It is a little more strict, so I like to mock-up a deployment in vanilla Kubernetes.

## Requirements

The steepest of requirements being that you need Kubernetes running -- actually getting Homer up and going afterwards is just a couple commands! Here's my short list:

* Kubernetes, 1.6.0 or greater (1.5 might work, I haven't tested it)
* Master node has git installed (and maybe your favorite text editor)

That's a short list, right? Well... Installing Kubernetes isn't that hard (and I've got the [ansible playbooks to do it](https://github.com/redhat-nfvpe/kube-centos-ansible)). But, we also need to have some persistent storage to use.

Why's that? Well... Containers are for the most part ephemeral in nature. They come, and they go, and they don't leave a lot of much around. We love them for that! They're very reproducible, and we love that. But -- with that we lose our data everytime they die. There's certain stuff we want to keep around with Homer -- especially: All of our database data. So we create persistent storage in order to keep it around. There's [many plugins for persistent volumes you can use with Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes), such as NFS, iSCSI, CephFS, Flocker (and proprietary stuff, too), etc. 

I **highly** recommend you follow my guide for [installing Kubernetes with  persistent volume storage backed by GlusterFS](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/). If you can follow through my guide successfully -- that will get you to exactly the place you need to be to follow the rest of this guide. It also puts you in a place that feasible for actually running in production -- the other option is to use "host path volumes", Kubernetes docs have a nice [tutorial on how to use host path volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) -- however, we lose a lot of the great value of Kubernetes when we use host path volumes -- they're not portable across hosts, so, they're effectly only good for a simple development use-case. If you do follow my guide -- I recommend that you stop before you actually create the MariaDB stuff. That way you don't have to clean it up (but you can leave it there and it won't cause any harm, tbh).

My guide on Kubernetes with GlusterFS backed persistent volumes also builds upon another one of my guide for [installing Kubernetes on CentOS](http://dougbtv.com//nfvpe/2017/02/16/kubernetes-1.5-centos/). Which may also be helpful.

Both of these guides use a CentOS 7.3 host, which we run Ansible playbooks against, to then run 4 virtual machines which comprise our Kubernetes 1.6.1 (at the time of writing) cluster.

If you're looking for something smaller (e.g. not a cluster), maybe you want to try [minikube](https://github.com/kubernetes/minikube).

## So, got your Kubernetes up and running?

Alright, if you've read this far, let's make sure we've got a working kubernetes, change your namespace, or if you're like me, I'll just mock this up in a default namespace. 

So, SSH into the master and just go ahead and check some basics....

```
[centos@kube-master ~]$ kubectl get nodes
[centos@kube-master ~]$ kubectl get pods
```

Everything looking to your liking? E.g. no errors and no pods running that you don't want running? Great, you're ready to rumble.

## Clone my fork, using the `k8s` branch.

Ok, next up, we're going to clone my fork of homer-docker, so go ahead and get that going...

```
[centos@kube-master ~]$ git clone -b k8s https://github.com/dougbtv/
[centos@kube-master ~]$ cd homer-docker/
```

Alright, now that we're there, let's take a small peek around.

First off in the root directoy there's a `k8s-docker-compose.yml`. It's a Docker compose file that's really only there for a single purpose -- and that's to build images from. The `docker-compose.yml` file that's there is for just a standard kind of deployment with just docker/docker-compose.

## Optional: Build your Docker images

If you want to build your own docker images and push them to a registry (say Dockerhub) -- now's the time to do that. It's completely optional -- if you don't, you'll just wind up pulling my images from Dockerhub, they're in the `dougbtv/*` namespace. Go ahead and skip ahead to the "persistent volumes" section if you don't want to bother with building your own.

So, first off you need Docker compose...

```
[centos@kube-master homer-docker]$ sudo /bin/bash -c 'curl -L https://github.com/docker/compose/releases/download/1.12.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose'
[centos@kube-master homer-docker]$ sudo chmod +x /usr/local/bin/docker-compose
[centos@kube-master homer-docker]$ sudo /usr/local/bin/docker-compose -v
docker-compose version 1.12.0, build b31ff33
```

And add it to your path if you wish.

Now -- go ahead and replace my namespace with your own. Replacing `YOURNAME` with, well, your own name (which is the namespace used in your registry for example)

```
[centos@kube-master homer-docker]$ find . -type f -print0 | xargs -0 sed -i 's/dougbtv/YOURNAME/g'
```

Now you can kick off a build.

```
[centos@kube-master homer-docker]$ sudo /usr/local/bin/docker-compose -f k8s-docker-compose.yml build
```

Now you'll have a bunch of images in your `docker images` list, and you can `docker login` and `docker push yourname/each-image` as you like. 

## Persistent volumes

Alright, now this is rather important, we're going to need persistent volumes to store our data in. So let's get those going.

I'm really hoping you followed my tutorial on [using Kubernetes with GlusterFS](http://dougbtv.com/nfvpe/2017/04/05/glusterfs-persistent/) because you'll have exactly the volumes we need. If you haven't -- I'm leaving this as an excersize for the reader to create [host path volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/), say if you're using minikube or otherwise. If you do choose that adventure, think about modifying my [glusterfs-volumes.yaml](https://github.com/redhat-nfvpe/kube-centos-ansible/blob/master/roles/glusterfs-kube-config/templates/glusterfs-volumes.yaml.j2) file.

During my tutorial where we created volumes, there's a file in cento's home @ `/home/centos/glusterfs-volumes.yaml` -- and we ran a `kubectl create -f /home/centos/glusterfs-volumes.yaml`. 

Once we ran that, we have volumes that are available to use, you can check them out with:

```
[centos@kube-master homer-docker]$ kubectl get pv
NAME               CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
gluster-volume-1   600Mi      RWO           Delete          Available             storage                  3h
gluster-volume-2   300Mi      RWO           Delete          Available             storage                  3h
gluster-volume-3   300Mi      RWO           Delete          Available             storage                  3h
gluster-volume-4   100Mi      RWO           Delete          Available             storage                  3h
gluster-volume-5   100Mi      RWO           Delete          Available             storage                  3h
```

Noting that in the above command `kubectl get pv` -- the pv means "persistent volumes". Once you have these volumes in your install -- you're good to proceed to the next steps.

## Drum roll please -- start a deploy of Homer!

Alright, now... with that in place there's just three steps we need to perform, and we'll look at the results of those after we run them. Those steps are:

* Make persistent volume claims (to stake a claim to the space in those volumes)
* Create service endpoints for the Homer services
* Start the pods to run the Homer containers

Alright, so go ahead and move yourself to the `k8s` directory in the clone you created earlier.

```
[centos@kube-master k8s]$ pwd
/home/centos/homer-docker/k8s
```

Now, there's 3-4 files here that really matter to us, go ahead and check them out if you so please.

These are the one's I'm talking about:

```
[centos@kube-master k8s]$ ls -1 *yaml
deploy.yaml
hepgen.yaml
persistent.yaml
service.yaml
```

The purpose of each of these files is...

* `persistent.yaml`: Defines our persistent volume claims.
* `deploy.yaml`: Defines which pods we have, and also configurations for them.
* `service.yaml`: Defines the exposed services from each pod.

Then there's `hepgen.yaml` -- but we'll get to that later!

Alright -- now that you get the gist of the lay of the land. Let's run each one.

### Changing some configuration options...

Should you need to change any options, they're generally environment variables and are in the `ConfigMap` section of the `deploy.yaml`. Some of those environment variables are really secrets, and it's an improvement that could be made to this deployment.

### Create Homer Persistent volume claims

Alright, we're going to need the persistent volume claims, so let's create those.

```
[centos@kube-master k8s]$ kubectl create -f persistent.yaml 
persistentvolumeclaim "homer-data-dashboard" created
persistentvolumeclaim "homer-data-mysql" created
persistentvolumeclaim "homer-data-semaphore" created
```

Now we can check out what was created.

```
[centos@kube-master k8s]$ kubectl get pvc
NAME                   STATUS    VOLUME             CAPACITY   ACCESSMODES   STORAGECLASS   AGE
homer-data-dashboard   Bound     gluster-volume-4   100Mi      RWO           storage        18s
homer-data-mysql       Bound     gluster-volume-2   300Mi      RWO           storage        18s
homer-data-semaphore   Bound     gluster-volume-5   100Mi      RWO           storage        17s
```

Great!

### Create Homer Services

Ok, now we need to create services -- which allows our containers to interact with one another, and us to interact the services they create.

```
[centos@kube-master k8s]$ kubectl create -f service.yaml 
service "bootstrap" created
service "cron" created
service "kamailio" created
service "mysql" created
service "webapp" created
```

Now, let's look at what's there.

```
[centos@kube-master k8s]$ kubectl get svc
NAME                CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
bootstrap           None             <none>        55555/TCP   6s
cron                None             <none>        55555/TCP   6s
glusterfs-cluster   10.107.123.112   <none>        1/TCP       23h
kamailio            10.105.142.140   <none>        9060/UDP    5s
kubernetes          10.96.0.1        <none>        443/TCP     1d
mysql               None             <none>        3306/TCP    5s
webapp              10.101.132.226   <none>        80/TCP      5s
```

You'll notice some familiar faces if you're used to deploying Homer with the `homer-docker` docker-compose file -- there's kamailio, mysql, the web app, etc. 

### Create Homer Pods

Ok, now -- we can create the actual containers to get Homer running.

```
[centos@kube-master k8s]$ kubectl create -f deploy.yaml 
configmap "env-config" created
job "bootstrap" created
deployment "cron" created
deployment "kamailio" created
deployment "mysql" created
deployment "webapp" created
```

Now, go ahead and watch them come up -- this could take a while during the phase where the images are pulled.

So go ahead and watch patiently...

    [centos@kube-master k8s]$ watch -n1 kubectl get pods --show-all

Wait until the `STATUS` for all pods is either `completed` or `running`. Should one of the pods fail, you might want to get some more information about it. Let's say the webapp isn't running; you could describe the pod, and get logs from the container with:

```
[centos@kube-master k8s]$ kubectl describe pod webapp-3002220561-l20v4
[centos@kube-master k8s]$ kubectl logs webapp-3002220561-l20v4
```

## Verify the backend -- generate some traffic with HEPgen.js

Alright, now that we have all the pods up -- we can create a hepgen job.

```
[centos@kube-master k8s]$ kubectl create -f hepgen.yaml 
job "hepgen" created
configmap "sample-hepgen-config" created
```

And go and watch that until the `STATUS` is `Completed` for the hepgen job.

    [centos@kube-master k8s]$ watch -n1 kubectl get pods --show-all

Now that it has run, we can verify that the calls are in the database, let's look at something simple-ish. Remember, the default password for the database is literally `secret`.

So go ahead and enter the command line for MySQL...

```
[centos@kube-master k8s]$ kubectl exec -it $(kubectl get pods | grep mysql | awk '{print $1}') -- mysql -u root -p
Enter password: 
```

And then you can check out the number of entries intoday `sip_capture_call_*` table. There should in theory be 3 entries here.

```
mysql> use homer_data;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables like '%0420%';
+-----------------------------------+
| Tables_in_homer_data (%0420%)     |
+-----------------------------------+
| isup_capture_all_20170420         |
| rtcp_capture_all_20170420         |
| sip_capture_call_20170420         |
| sip_capture_registration_20170420 |
| sip_capture_rest_20170420         |
| webrtc_capture_all_20170420       |
+-----------------------------------+
6 rows in set (0.01 sec)

mysql> SELECT COUNT(*) FROM sip_capture_call_20170420;
+----------+
| COUNT(*) |
+----------+
|        3 |
+----------+
1 row in set (0.00 sec)
```

It's all there! That means our backend is generally working. But... That's the hard dirty work for Homer, we want to get into the good stuff -- some visualization of our data, so let's move on to the front-end.

## Expose the front-end

Alright in theory now you can look at `kubectl get svc` and see the service for the webapp, and visit that URL. 

But, following with my tutorial, if you've run these in VMs on a CentOS host, well... you have a little more to do to expose the front-end. This is also (at least somewhat) similiar to how you'd expose an external IP address to access this in a more-like-production setup.

So, let's go ahead and change the service for the webapp to match the external IP address of the master. 

Note, you'll see it has an internal IP address assigned by your CNI networking IPAM.

```
[centos@kube-master k8s]$ kubectl get svc | grep -Pi "^Name|webapp"
NAME                CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
webapp              10.101.132.226   <none>        80/TCP      32m
```

Given that, right from the master (or likely anywhere on Kube nodes) you could just `curl 10.101.132.226` and there's your dashboard, but, man it's hard to navigate the web app using curl ;)

So let's figure out the IP address of our host. Mine is in the `192.168.122`range and yours will be too if you're using my VM method here.

```
[centos@kube-master k8s]$ kubectl delete svc webapp
service "webapp" deleted
[centos@kube-master k8s]$ ipaddr=$(ip a | grep 192 | awk '{print $2}' | perl -pe 's|/.+||')
[centos@kube-master k8s]$ kubectl expose deployment webapp --port=80 --target-port=80 --external-ip $ipaddr
service "webapp" exposed
```

Now you'll see we have an external address, so anyone who can access `192.168.122.14:80` can see this.

```
[centos@kube-master k8s]$ kubectl get svc | grep -Pi "^Name|webapp"
NAME                CLUSTER-IP       EXTERNAL-IP      PORT(S)     AGE
webapp              10.103.140.63    192.168.122.14   80/TCP      53s
```

However if you're using my setup, you might have to tunnel traffic from your desktop to the virtual machine host in order to do that. So, I did so with something like:

```
ssh root@virtual_machine_host -L 8080:192.168.122.14:80
```

Now -- I can type in `localhost:8080` in my browser, and... Voila! There is Homer!

Remember to login with username `admin` and password `test123`.

Change your date to "today" and hit search, and you'll see all the information we captured from running HEPgen.

And there... You have it! Now you can wrangle in what's going on with your Kubernetes VoIP platform :)