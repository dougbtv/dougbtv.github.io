---
author: dougbtv
comments: true
date: 2016-08-03 17:00:00-05:00
layout: post
slug: koko-service-chaining
title: Service Chaining in Container using Koko & Koro
category: nfvpe
---

## Service chaining

Next we're going to do some "service chaining". It's not exactly "service function chaining" (SFC) -- we can let [sdxcentral define that for you](https://www.sdxcentral.com/sdn/network-virtualization/definitions/what-is-network-service-chaining/). From what I understand is that pure SFC uses a "network service header" ([which you can see here from IETF](https://datatracker.ietf.org/doc/draft-ietf-sfc-nsh/)) to help perform dynamic routing. This doesn't use those headers, so I will refer to it as simply "service chaining". In fact... We're going to perform a series of steps here that are quite manual, but, to demonstrate what you may be able to automate in the future -- and my associate Tomofumi has some machinations in the works to do such things. We'll cover those later.

Now that we've establashed we're going to chain some services together -- let's go ahead and actually chain 'em up!


## ntopng notes

I've created an [ntopng](http://www.ntop.org/products/traffic-analysis/ntop/) community container image, available from [this Dockerfile in a gist](https://gist.github.com/dougbtv/52ef2c92e87c3398a8752cf0ebc6419e).

```
$ docker run -dt -p 3000:3000 --privileged --name=ntop --entrypoint=/bin/bash dougbtv/ntopng
$ docker run --name test1 --net=none -dt dougbtv/centos-network sleep 2000000
$ docker run --name test2 --net=none -dt dougbtv/centos-network sleep 2000000
$ ./gocode/bin/koko -d test1,link1,10.0.1.1/24 -d ntop,link2,10.0.1.2/24
$ ./gocode/bin/koko -d ntop,link3,10.0.2.1/24 -d test2,link4,10.0.2.2/24
```

That was... a failboat. The [bridge feature](http://www.ntop.org/ndpi/how-to-enforce-layer-7-traffic-policies-using-ntopng/) that I wanted to use is not a community feature.

Falling back to "just a firewall"

## iptables notes.

```
$ docker pull vimagick/iptables
$ docker run --name=iptables -dt --privileged -e 'TCP_PORTS=80,443' -e 'UDP_PORTS=53' -e 'RATE=4mbit' -e 'BURST=4kb' vimagick/iptables:latest
$ docker run --name test1 --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ docker run --name test2 --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ ./gocode/bin/koko -d test1,link1,10.0.1.1/24 -d iptables,link2,10.0.1.2/24
$ ./gocode/bin/koko -d iptables,link3,10.0.2.1/24 -d test2,link4,10.0.2.2/24
```

Then, you need default routes on both `test1` and `test2`, like:

```
[root@0f821a241d24 /]# ip route add default via 10.0.1.2 dev link1
```

And the iptables container needs to have ip forwarding...

```
[root@koko1 centos]# docker exec -it iptables /bin/sh
/ # echo 1 > /proc/sys/net/ipv4/ip_forward
```

Then you should be able to ping `10.0.2.2` from `test1`.

Now let's block icmp, to make sure iptables is working, needs to go into the `FORWARD` table.

```
/ # iptables -A FORWARD -p icmp  -j DROP
```

And you can remove that too...

```
/ # iptables delete -j FORWARD 1
```

## Service chain with koro

Now let's do the same thing but with koro.

```
$ docker run --name=iptables -dt --privileged -e 'TCP_PORTS=80,443' -e 'UDP_PORTS=53' -e 'RATE=4mbit' -e 'BURST=4kb' vimagick/iptables:latest
$ docker run --name test1 --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ docker run --name test2 --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ ./gocode/bin/koko -d test1,link1 -d iptables,link2
$ ./gocode/bin/koko -d iptables,link3 -d test2,link4
```

Alright, now, you've gotta still set ip forwarding on the iptables container.

```
[root@koko1 centos]# docker exec -it iptables /bin/sh
/ # echo 1 > /proc/sys/net/ipv4/ip_forward
```

We've got links now, but, no ip addressing. Koro should be able to fix this up for us.

This adds the addresses...

```
$ ./gocode/bin/koro docker test1 address add 10.0.1.1/24 dev link1
$ ./gocode/bin/koro docker iptables address add 10.0.1.2/24 dev link2
$ ./gocode/bin/koro docker iptables address add 10.0.2.1/24 dev link3
$ ./gocode/bin/koro docker test2 address add 10.0.2.2/24 dev link4
```

Let's add a default route to test1 & 2.

```
$ ./koro docker test1 route add default via 10.0.1.2 dev link1
$ ./koro docker test2 route add default via 10.0.2.1 dev link4
```

With those in place, we can now ping across the containers.

```
$ docker exec -it test1 ping -c 5 10.0.2.2
```

Alright, and now... we'll take those down. (This kills all containers running on your host, btw.)

```
$ docker kill $(docker ps -aq)
$ docker rm $(docker ps -aq)
```

## Setting up a service chain.

So let's go ahead and make a service chain, it'll look like...

```
functions: client --> firewall --> router --> webserver
```

So let's spin up all the pieces that we need.

Pull my `dougbtv/pickle-nginx`, we'll use that.

```
$ docker pull dougbtv/pickle-nginx
```

Now, let's run all the containers.

```
$ docker run --name client --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ docker run --name=firewall -dt --privileged -e 'TCP_PORTS=80,443' -e 'UDP_PORTS=53' -e 'RATE=4mbit' -e 'BURST=4kb' vimagick/iptables:latest
$ docker run --name router --privileged --net=none -dt dougbtv/centos-network sleep 2000000
$ docker run -dt --net=none --name webserver dougbtv/pickle-nginx
```

And run a `docker ps` to make sure they're all running.

Ok, these need a bit of grooming. Firstly, we need IP forwarding on the firewall and router.

```
$ docker exec -it firewall /bin/sh -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
$ docker exec -it router /bin/sh -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
```

Great. Now we can create koko links between all the containers. That's three veth pairs...

```
$ ./gocode/bin/koko -d client,link1 -d firewall,link2
$ ./gocode/bin/koko -d firewall,link3 -d router,link4
$ ./gocode/bin/koko -d router,link5 -d webserver,link6
```

And now we'll add addresses to them all.

```
$ ./gocode/bin/koro docker client address add 10.0.1.1/24 dev link1
$ ./gocode/bin/koro docker firewall address add 10.0.1.2/24 dev link2
$ ./gocode/bin/koro docker firewall address add 10.0.2.1/24 dev link3
$ ./gocode/bin/koro docker router address add 10.0.2.2/24 dev link4
$ ./gocode/bin/koro docker router address add 10.0.3.1/24 dev link5
$ ./gocode/bin/koro docker webserver address add 10.0.3.2/24 dev link6
```

And we're going to need some more routing.

```
[root@koko1 centos]# ./gocode/bin/koro docker client route add default via 10.0.1.2 dev link1
[root@koko1 centos]# ./gocode/bin/koro docker webserver route add default via 10.0.3.1 dev link6
[root@koko1 centos]# ./gocode/bin/koro docker firewall route add 10.0.3.0/24 via 10.0.2.2 dev link3
[root@koko1 centos]# ./gocode/bin/koro docker router route add 10.0.1.0/24 via 10.0.2.1 dev link4
```

Check all the routing.

```
[root@koko1 centos]# docker exec -it client ip route
default via 10.0.1.2 dev link1 
10.0.1.0/24 dev link1  proto kernel  scope link  src 10.0.1.1 

[root@koko1 centos]# docker exec -it firewall ip route
default via 172.17.0.1 dev eth0 
10.0.1.0/24 dev link2 proto kernel scope link src 10.0.1.2 
10.0.2.0/24 dev link3 proto kernel scope link src 10.0.2.1 
10.0.3.0/24 via 10.0.2.2 dev link3 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2 

[root@koko1 centos]# docker exec -it router ip route
10.0.1.0/24 via 10.0.2.1 dev link4 
10.0.2.0/24 dev link4  proto kernel  scope link  src 10.0.2.2 
10.0.3.0/24 dev link5  proto kernel  scope link  src 10.0.3.1 

[root@koko1 centos]# docker exec -it webserver ip route
default via 10.0.3.1 dev link6 
10.0.3.0/24 dev link6  proto kernel  scope link  src 10.0.3.2 
```

Now we have a service chain! Huzzah! You can curl the nginx.

```
[root@koko1 centos]# docker exec -it client /bin/bash -c 'curl -s 10.0.3.2 | grep -i pickle'
<title>This is pickle-nginx</title>
```

## Removing an item and fixing the links

Let's cause some [chaos, some mass confusion](https://www.youtube.com/watch?v=qJHEI8fJdiM). It's all well and good we have these four pieces all setup together.

However, the reality is... Something is going to happen. In the real world -- everything is broken. To emulate that let's create this scenario -- the firewall goes down. In a more realistic scenario, this pod will be recreated. For this demonstration we're just going to let it be gone, and we'll just create new links with koko directly to the router, and then re-route

```
[root@koko1 centos]# docker kill firewall
```

That should do it. Alright now we can't run our same curl, it fails.

```
[root@koko1 centos]# docker exec -it client /bin/bash -c 'curl 10.0.3.2'
curl: (7) Failed to connect to 10.0.3.2: Network is unreachable
```

We can use koko & koro to fix this up for us. Let's create some new interfaces with koko. We'll also just use a new subnet for this connection (we could finesse the existing, but, this is a couple steps less).

Go ahead and create that veth pair.

```
$ ./gocode/bin/koko -d client,link7 -d router,link8
```

Now, we'll need some IP addresses, too.

```
$ ./gocode/bin/koro docker client address add 10.0.4.1/24 dev link7
$ ./gocode/bin/koro docker router address add 10.0.4.2/24 dev link8
```

And we have to fix the client containers default route. We don't have to delete the existing default route because it went down with the interface -- since a veth is a pair. (In a vxlan setup, we'd have to otherwise detect the failure and provide some cleanup), so all we have to do is add a route.

```
./gocode/bin/koro docker client route add default via 10.0.4.2 dev link7
```

And -- we're back in business, you can curl the `pickle-nginx` again.

```
[root@koko1 centos]# docker exec -it client /bin/bash -c 'curl -s 10.0.3.2 | grep -i pickle'
<title>This is pickle-nginx</title>
```

## In closing.

Using the basics from this technique for a failed service in a container you could make a number of other operations that would use the same basics, e.g. other failure modes (container that is died is replaced with a new one), or extensions of the service chain, say... Adding a DPI container somewhere in the chain.

The purpose of this is to show the steps manually that could be taken automatically -- by say a CNI plugin for example. That could make these changes automatically and much more quickly than us lowly humans can make them by punching commands in a terminal.

## Tomo's VPP notes

His installation

```
yum -y install git
git clone https://gerrit.fd.io/r/vpp
cd vpp
make install-dep
make bootstrap
make build
make run
```

> To recognize interface (e.g. eth0/eth1), you need to modify /etc/vpp/startup.conf as following:
https://wiki.fd.io/view/VPP/How_To_Connect_A_PCI_Interface_To_VPP
vpp recognizes virtio interface of qemu, but some interface is not supported yet (due to dpdk), so please take care of baremetal case.

---

creating vpp vxlan tunnels

```
# Each will output name of the created tunnel
create vxlan tunnel src 10.1.1.1 dest 10.1.1.11 vni 11
create vxlan tunnel src 10.1.1.2 dest 10.1.1.12 vni 12
```

creating koko devices...

```
# box a
koko -d test,link1,192.168.1.1/24 -x eth1,10.1.1.1,11
# box b
koko -d test,link1,192.168.1.2/24 -x eth1,10.1.1.1,12
```

Create the vpp cross connections

```
set interface l2 xconnect vxlan_tunnel0 vxlan_tunnel1
set interface l2 xconnect vxlan_tunnel1 vxlan_tunnel0
```

## Tomo's SFC notes

```
<tohayash> just a info about the term "SFC (service function chaining)". 
<dougbtv> sure
<dougbtv> I appreciate any input and assistance there, for sure
<tohayash> previously Feng said that "SFC" sometimes feels "dynamic routing by service header", so from his definition, our solution is not based SFC.
<tohayash> If someone thinking "SFC is kind of service chaining", our solution is following their definition.
<dougbtv> ahhhh ha, it's a more generic example of just "chaining some things together" -- and not "EXACTLY service function chaining"
<dougbtv> really good to know!!!
<tohayash> so we may use service chaining or other terms in blog to avoid misunderstanding...
<tohayash> yeah...
<dougbtv> thank you :) that's an excellent pointer
<tohayash> yeah, sometimes SFC means "SFC-NSH" only, so we may need to use other terms for NSH fundamentalist :)
<tohayash> anyway, just about it. have a good day, doug!
<dougbtv> Really good call Tomo, the fundamentalists do get picky. Thanks much again! Have a nice night, catch you soon
```

