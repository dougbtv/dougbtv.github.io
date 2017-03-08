---
author: dougbtv
comments: true
date: 2017-03-08 12:01:00-05:00
layout: post
slug: kuryr-kubernetes
title: Kuryr-Kubernetes will knock your socks off!
category: nfvpe
---

Seeing [kuryr-kubernetes](https://github.com/openstack/kuryr-kubernetes) in action in my "[Dr. Octagon NFV laboratory](https://github.com/dougbtv/droctagon-ansible)" has got me [feeling that barefoot feeling](https://www.youtube.com/watch?v=kuprjBXuRls) -- and henceforth has completely knocked my socks off. Kuryr-Kubernetes provides Kubernetes integration with OpenStack networking, and today we'll walk through the steps so you can get your own instance up of it up and running so you can check it out for yourself. We'll spin up kuryr-kubernetes with devstack, create some pods and a VM, inspect Neutron and verify the networking is working a charm.

As usual with these blog posts, I'm kind of [standing on the shoulders of giants](https://en.wikipedia.org/wiki/Standing_on_the_shoulders_of_giants). I was able to get some great exposure to [kuryr-kubernetes through Luis Tomas's blog post](https://ltomasbo.wordpress.com/2017/01/29/side-by-side-and-nested-kubernetes-and-openstack-deployment-with-kuryr/). And then a lot of the steps here you'll find familiar from [this OpenStack superuser blog post](http://superuser.openstack.org/articles/networking-kubernetes-kuryr/). Additionally, I always wind up finding a good show-stopper or two, and Antoni Segura Puimedon ([celebdor](https://github.com/celebdor)) was a huge help in diagnosing my setup, which I greatly appreciated.

## Requirements

You might be able to do this with a VM, but, you'll need some kind of nested virtualization -- because we're going to spin up a VM, too. In my case, I used baremetal and the machine is likely overpowered (48 gig RAM, 16 cores, 1TB spinning disk). I'd recommend no less than 4-8 gigs of RAM and at least a few cores, and maybe 20-40 gigs free (which is still overkill)

One requirement that's basically for sure is a CentOS 7.3 (or later) install somewhere. I assume you've got this setup. Also, make sure it's pretty fresh because I've run into problems with devstack where I tried to put it on an existing machine and it fought with say an existing Docker install.

That box needs git, and maybe your favorite text editor (and I use screen).

## Get your devstack up and kickin'

The gist here is that we'll clone devstack, setup the stack user, create a `local.conf` file, and then kick off the stack.sh

So here's where we clone devstack, use it to create a `stack` user, and move the devstack clone into the stack user's home and then assume that user.

```
[root@droctagon3 ~]# git clone https://git.openstack.org/openstack-dev/devstack
[root@droctagon3 ~]# cd devstack/
[root@droctagon3 devstack]# ./tools/create-stack-user.sh 
[root@droctagon3 devstack]# cd ../
[root@droctagon3 ~]# mv devstack/ /opt/stack/
[root@droctagon3 ~]# chown -R stack:stack /opt/stack/
[root@droctagon3 ~]# su - stack
[stack@droctagon3 ~]$ pwd
/opt/stack
```

Ok, now that we're there, let's create a `local.conf` to parameterize our devstack deploy. You'll note that my config is a portmanteau of Luis' and from the superuser blog post. I've left in my comments even so you can check it out and compare against the references. Go ahead and put this in with an echo heredoc or your favorite editor, here's mine:

```
[stack@droctagon3 ~]$ cd devstack/
[stack@droctagon3 devstack]$ pwd
/opt/stack/devstack
[stack@droctagon3 devstack]$ cat local.conf 
[[local|localrc]]

LOGFILE=devstack.log
LOG_COLOR=False

# HOST_IP=CHANGEME
# Credentials
ADMIN_PASSWORD=pass
MYSQL_PASSWORD=pass
RABBIT_PASSWORD=pass
SERVICE_PASSWORD=pass
SERVICE_TOKEN=pass
# Enable Keystone v3
IDENTITY_API_VERSION=3

# Q_PLUGIN=ml2
# Q_ML2_TENANT_NETWORK_TYPE=vxlan

# LBaaSv2 service and Haproxy agent
enable_plugin neutron-lbaas \
 git://git.openstack.org/openstack/neutron-lbaas
enable_service q-lbaasv2
NEUTRON_LBAAS_SERVICE_PROVIDERV2="LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default"

enable_plugin kuryr-kubernetes \
 https://git.openstack.org/openstack/kuryr-kubernetes refs/changes/45/376045/12

enable_service docker
enable_service etcd
enable_service kubernetes-api
enable_service kubernetes-controller-manager
enable_service kubernetes-scheduler
enable_service kubelet
enable_service kuryr-kubernetes

# [[post-config|/$Q_PLUGIN_CONF_FILE]]
# [securitygroup]
# firewall_driver = openvswitch
``` 

Now that we've got that set. Let's just at least take a look at one parameters. The one in question is:

```
enable_plugin kuryr-kubernetes \
 https://git.openstack.org/openstack/kuryr-kubernetes refs/changes/45/376045/12
```

You'll note that this is version pinned. I ran into a bit of a hitch that Toni helped get me out of. And we'll use that work-around in a bit. [There's a patch that's coming along](https://review.openstack.org/#/c/442794/1) that should fix this up. I didn't have luck with it yet, but, just submitted the evening before this blog post.

Now, let's run that devstack deploy, I run mine in a screen, that's optional for you, but, I don't wanna have connectivity lost during it and wonder "what happened?".

```
[stack@droctagon3 devstack]$ screen -S devstack
[stack@droctagon3 devstack]$ ./stack.sh 
```

Now, relax... This takes ~50 minutes on my box. 

## Verify the install and make sure the kubelet is running

Alright, that should finish up and show you some timing stats and some URLs for your devstack instances.

Let's just mildly verify that things work.

```
[stack@droctagon3 devstack]$ source openrc 
[stack@droctagon3 devstack]$ nova list
+----+------+--------+------------+-------------+----------+
| ID | Name | Status | Task State | Power State | Networks |
+----+------+--------+------------+-------------+----------+
+----+------+--------+------------+-------------+----------+
```

Great, so we have some stuff running at least. But, what about Kubernetes?

It's likely almost there.

```
[stack@droctagon3 devstack]$ kubectl get nodes
```

That's going to be empty for now. It's because the kubelet isn't running. So, open the devstack "screens" with:

```
screen -r
```

Now, tab through those screens, hit `Ctrl+a` then `n`, and it will go to the next screen. Keep going until you get to the `kubelet` screen. It will be at the lower left hand size and/or have an `*` next to it.

It will likely be a screen with "just a prompt" and no logging. This is because the kubelet fails to run in this iteration, but, we can work around it.

First off, get your IP address, mine is on my interface `enp1s0f1` so I used `ip a` and got it from there. Now, put that into the below command where I have `YOUR_IP_HERE`

Issue this command to run the kubelet:

```
sudo /usr/local/bin/hyperkube kubelet\
        --allow-privileged=true \
        --api-servers=http://YOUR_IP_HERE:8080 \
        --v=2 \
        --address='0.0.0.0' \
        --enable-server \
        --network-plugin=cni \
        --cni-bin-dir=/opt/stack/cni/bin \
        --cni-conf-dir=/opt/stack/cni/conf \
        --cert-dir=/var/lib/hyperkube/kubelet.cert \
        --root-dir=/var/lib/hyperkube/kubelet
```

Now you can detach from the screen by hitting `Ctrl+a` then `d`. You'll be back to your regular old prompt.

Let's list the nodes...

```
[stack@droctagon3 demo]$ kubectl get nodes
NAME         STATUS    AGE
droctagon3   Ready     4s
```

And you can see it's ready to rumble.

## Build a demo container

So let's build something to run here. We'll use the same container in a pod as shown in the superuser article.

Let's create a python script that runs an http server and will report the hostname of the node it runs on (in this case when we're finished, it will report the name of the pod in which it resides)

So let's create those two files, we'll put them in a "demo" dir.

```
[stack@droctagon3 demo]$ pwd
/opt/stack/devstack/demo
```

Now make the `Dockerfile`:

```
[stack@droctagon3 demo]$ cat Dockerfile 
FROM alpine
RUN apk add --no-cache python bash openssh-client curl
COPY server.py /server.py
ENTRYPOINT ["python", "server.py"]
```

And the `server.py`

```
[stack@droctagon3 demo]$ cat server.py 
import BaseHTTPServer as http
import platform

class Handler(http.BaseHTTPRequestHandler):
  def do_GET(self):
    self.send_response(200)
    self.send_header('Content-Type', 'text/plain')
    self.end_headers()
    self.wfile.write("%s\n" % platform.node())

if __name__ == '__main__':
  httpd = http.HTTPServer(('', 8080), Handler)
  httpd.serve_forever()
```

And kick off a Docker build.

```
[stack@droctagon3 demo]$ docker build -t demo:demo .
```

## Kick up a Pod

Now we can launch a pod given that, we'll even skip the step of making a yaml pod spec since this is so simple.

```
[stack@droctagon3 demo]$ kubectl run demo --image=demo:demo
```

And in a few seconds you should see it running...

```
[stack@droctagon3 demo]$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
demo-2945424114-pi2b0   1/1       Running   0          45s
```

## Kick up a VM

Cool, that's kind of awesome. Now, let's create a VM.

So first, download a cirros image.

```
[stack@droctagon3 ~]$ curl -o /tmp/cirros.qcow2 http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

Now, you can upload it to glance.

```
glance image-create --name cirros --disk-format qcow2  --container-format bare  --file /tmp/cirros.qcow2 --progress
```

And we can kick off a pretty basic nova instance, and we'll look at it a bit.

```
[stack@droctagon3 ~]$ nova boot --flavor m1.tiny --image cirros testvm
[stack@droctagon3 ~]$ openstack server list -c Name -c Networks -c 'Image Name'
+--------+---------------------------------------------------------+------------+
| Name   | Networks                                                | Image Name |
+--------+---------------------------------------------------------+------------+
| testvm | private=fdae:9098:19bf:0:f816:3eff:fed5:d769, 10.0.0.13 | cirros     |
+--------+---------------------------------------------------------+------------+
```

## Kuryr magic has happened! Let's see what it did.

So, now Kuryr has performed some cool stuff, we can see that it created a Neutron port for us.

```
[stack@droctagon3 ~]$ openstack port list --device-owner kuryr:container -c Name
+-----------------------+
| Name                  |
+-----------------------+
| demo-2945424114-pi2b0 |
+-----------------------+
[stack@droctagon3 ~]$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
demo-2945424114-pi2b0   1/1       Running   0          5m
```

You can see that the port name is the same as the pod name -- cool!

And that pod has an IP address on the same subnet as the nova instance. So let's inspect that.

```
[stack@droctagon3 ~]$ pod=$(kubectl get pods -l run=demo -o jsonpath='{.items[].metadata.name}')
[stack@droctagon3 ~]$ pod_ip=$(kubectl get pod $pod -o jsonpath='{.status.podIP}')
[stack@droctagon3 ~]$ echo Pod $pod IP is $pod_ip
Pod demo-2945424114-pi2b0 IP is 10.0.0.4
```

## Expose a service for the pod we launched

Ok, let's go ahead and expose a service for this pod. We'll expose it and see what the results are.

```
[stack@droctagon3 ~]$ kubectl expose deployment demo --port=80 --target-port=8080
service "demo" exposed
[stack@droctagon3 ~]$ kubectl get svc demo
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
demo      10.0.0.84    <none>        80/TCP    13s
[stack@droctagon3 ~]$ kubectl get endpoints demo
NAME      ENDPOINTS       AGE
demo      10.0.0.4:8080   1m
```

And we have an LBaaS (load balancer as a service) which we can inspect with neutron...

```
[stack@droctagon3 ~]$ neutron lbaas-loadbalancer-list -c name -c vip_address -c provider
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+------------------------+-------------+----------+
| name                   | vip_address | provider |
+------------------------+-------------+----------+
| Endpoints:default/demo | 10.0.0.84   | haproxy  |
+------------------------+-------------+----------+
[stack@droctagon3 ~]$ neutron lbaas-listener-list -c name -c protocol -c protocol_port
[stack@droctagon3 ~]$ neutron lbaas-pool-list -c name -c protocol
[stack@droctagon3 ~]$ neutron lbaas-member-list Endpoints:default/demo:TCP:80 -c name -c address -c protocol_port
[stack@droctagon3 ~]$ neutron lbaas-member-list Endpoints:default/demo:TCP:80 -c name -c address -c protocol_port
```

## Scale up the replicas

You can now scale up the number of replicas of this pod, and Kuryr will follow along in suit. Let's do that now.

```
[stack@droctagon3 ~]$ kubectl scale deployment demo --replicas=2
deployment "demo" scaled
[stack@droctagon3 ~]$ kubectl get pods
NAME                    READY     STATUS              RESTARTS   AGE
demo-2945424114-pi2b0   1/1       Running             0          14m
demo-2945424114-rikrg   0/1       ContainerCreating   0          3s
```

We can see that more ports were created... 

```
[stack@droctagon3 ~]$ openstack port list --device-owner kuryr:container -c Name -c 'Fixed IP Addresses'
[stack@droctagon3 ~]$ neutron lbaas-member-list Endpoints:default/demo:TCP:80 -c name -c address -c protocol_port
```

## Verify connectivity

Now -- as if the earlier goodies weren't fun, this is the REAL fun part. We're going to enter a pod, e.g. via `kubectl exec` and we'll go ahead and check out that we can reach the pod from the pod, and the VM from the pod, and the exposed service (and henceforth both pods) from the VM. 

Let's do it! So go and exec the pod, and we'll give it a cute prompt so we know where we are since we're about to enter the rabbit hole.

```
[stack@droctagon3 ~]$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
demo-2945424114-pi2b0   1/1       Running   0          21m
demo-2945424114-rikrg   1/1       Running   0          6m
[stack@droctagon3 ~]$ kubectl exec -it demo-2945424114-pi2b0 /bin/bash
bash-4.3# export PS1='[user@pod_a]$ '
[user@pod_a]$ 
```

Before you continue on -- you might want to note some of the IP addresses we showed earlier in this process. Collect those or chuck 'em in a note pad and we can use them here.

Now that we have that, we can verify our service locally.

```
[user@pod_a]$ curl 127.0.0.1:8080
demo-2945424114-pi2b0
```

And verify it with the pod IP

```
[user@pod_a]$ curl 10.0.0.4:8080
demo-2945424114-pi2b0
```

And verify we can reach the other pod

```
[user@pod_a]$ curl 10.0.0.11:8080
demo-2945424114-rikrg
```

Now we can verify the service, note how you get different results from each call, as it's load balanced between pods.

```
[user@pod_a]$ curl 10.0.0.84
demo-2945424114-pi2b0
[user@pod_a]$ curl 10.0.0.84
demo-2945424114-rikrg
```

Cool, how about the VM? We should be able to ssh to it since it uses the default security group which is pretty wide open. Let's ssh to that (reminder, password is `cubswin:)`) and also set the prompt to look cute.

```
[user@pod_a]$ ssh cirros@10.0.0.13
The authenticity of host '10.0.0.13 (10.0.0.13)' can't be established.
RSA key fingerprint is SHA256:Mhz/s1XnA+bUiCZxVc5vmD1C6NoeCmOmFOlaJh8g9P8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.13' (RSA) to the list of known hosts.
cirros@10.0.0.13's password: 
$ export PS1='[cirros@vm]$ '
[cirros@vm]$ 
```

Great, so that definitely means we can get to the VM from the pod. But, let's go and curl that service!

```
[cirros@vm]$ curl 10.0.0.84
demo-2945424114-pi2b0
[cirros@vm]$ curl 10.0.0.84
demo-2945424114-rikrg
```

Voila! And that concludes our exploration of kuryr-kubernetes for today. Remember that you can find the Kuryr crew on the openstack mailing lists, and also in Freenode @ `#openstack-kuryr`.