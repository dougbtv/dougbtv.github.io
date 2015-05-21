---
author: dougbtv
comments: true
date: 2015-05-21 09:08:00-05:00
layout: post
slug: high-availability-asterisk-coreos
title: High Availability Asterisk using Docker & CoreOS with etcd
---

After I created my [Docker project for using Asterisk with Docker](https://github.com/dougbtv/docker-asterisk) -- I couldn't leave well enough alone. I wanted to create an Asterisk eco-system that had a number of features that just a stand-alone Asterisk couldn't do for me (while it definitely provides the life-blood of my VoIP systems as it is!)

The things that I really want in an Asterisk setup are:
* High-availability
* Scalability
* Visibility (of SIP traffic)

I've since created a set of tools, [available in the same docker-asterisk git repo](https://github.com/dougbtv/docker-asterisk), to accomplish this. Take a look for yourself in the [high-availability directory](https://github.com/dougbtv/docker-asterisk/tree/master/high-availability) in your clone.

The advantage of this project is that it's really easy to setup this eco-system, with the potential for tens or hundreds of boxen -- with great ease. [Choosing Docker](https://www.docker.com/whatisdocker/) lets us leverage the advantage of containerizing our applications and make them easy to test in development, and deploy to production (with assurance that we have developed in environments congruent with production). [Choosing CoreOS](https://coreos.com/using-coreos/) lets us run those Docker containers easily, and leverages CoreOS's `etcd` which allows for service discovery -- allowing us to spin up more and more nodes in our cluster -- and let them discover one another.

You could use this setup for a very manageable, and scalable high-availability Asterisk setup -- one that's easily deployable in the cloud (or the closet). One could also customize the setup to dynmically allocate or deallocate more Asterisk boxes during their busy hour -- if paying per cycle with a cloud provider, it's plausible dynamically spinning boxes up and down could save you some considerable "[skrilla](http://www.urbandictionary.com/define.php?term=skrilla)" (that's money).

If you aren't familiar with the current wave/trend of containerization, and it's advantages, check out [this recent InformationWeek article](http://www.informationweek.com/cloud/platform-as-a-service/linux-container-operating-systems-thin-is-in/a/d-id/1320478) on Linux containers.

With the setup provided here, any given box can go down, and calls will continue to flow into the platform. While it does not have a replicated state machine between Asterisk boxes themselves (however, one may be able to achieve the same with 1:1 hardware redundancy using the virtualization platform of their choice), the philosophy is to have a highly-available load-balancer (which we use Kamailio for) that can dynamically balance load between N-number of Asterisk boxes, and to keep a low call volume on each Asterisk box so that in the case of catastrophic failure of a given Asterisk box, you lose the least amount of calls. The load balancer is a place where we could have a single point of failure, but, this is solved by having identical configurations on 2 (or more) load-balancers and using a virtual IP to fail over between the two (or more) Kamailio load balancers.

It also considers the regular maintenance you might do, and will gracefully drain calls from your Asterisk boxes if you issue at the Asterisk*CLI, for example, `core shutdown gracefully`. There are also other ways to put your Asterisk boxes into a maintenance mode, that we will cover down the road.

This article covers the methodology and philosophy behind the choices used in the `docker-asterisk` project. I will later follow this article with a deep-dive and tutorial into the components behind it and show you just how easy it is to stand-up a scalable HA cluster of Asterisk boxes.

The core application technologies I've chosen for telephony to accomplish this are:
* Asterisk (naturally! For a media server, IVR, PBX or feature server)
* Kamailio (for load balancing Asterisk)
* [Homer](http://www.sipcapture.org/) & [captagent](https://github.com/sipcapture/captagent) (for visibility of SIP traffic)

The general gist is to deploy containerized editions of each the core applications under the technology stack that follows, they're configured in a specific way in order to interoperate.

The platform technology stack I've chosen consists of:

* [Docker](https://www.docker.com/whatisdocker/) (for containerization)
* [CoreOS](https://coreos.com/) (for scalability)
* [etcd](https://coreos.com/etcd/) (for service discovery)
* [keepalived](http://www.keepalived.org/) (for virtual IPs and fail-over)

I wound up choosing [CoreOS](https://coreos.com/) to use as the base operating system, which is great at running Docker containers, and comes with etcd -- a great tool for service discovery (which helps our scalability features, quite a bit)

This is a good time to show you an overview of the platform stack:

![Platform Stack](http://i.imgur.com/2zTAII9.png)

We can see that on our host, runs CoreOS as the base operating system, and etcd is included with the CoreOS installation, and will dynamically share service discovery information with the whole CoreOS cluster. CoreOS will be running Docker proper, and on top of that layer we run each of our Docker containers. 

Most of the names of the Docker containers should be familiar to you, such as Asterisk, Kamailio & Homer (homer uses a [LAMP stack](http://en.wikipedia.org/wiki/LAMP_%28software_bundle%29), hence the multide of containers [the four on the right]).

One that is likely new to you is the [kamailio-etcd-dispatcher](https://github.com/dougbtv/kamailio-etcd-dispatcher) container. `kamailio-etcd-dispatcher` is a custom open-source application that I created that allows you to spin up more and more Asterisk boxes and have the service to be discovered by our Kamailio load balancer. In one role it announces it's presence, and the companion role it updates (and applies) a new `dispatcher.list` for Kamailio. (By default, it evenly load balances each box, however, it supports setting meta data that allows you to set a custom distribution if it suits your needs). This application is written in Node.js -- but don't sweat it if you're not running node.js on your platform currently -- since it's containerized, you really can just "let go" of that configuration and let it happen in the container. 

Let's take a look at the neworking:

![Networking](http://i.imgur.com/G4yu1Pq.png)

Let's read it left to right, and you'll see that we have an ITSP who's talking to a virtual IP. This virtual IP (VIP) is handled by Keepalived, and is virtually shared between two Kamailio boxes. A SIP message will be transmitted to this VIP -- and the Kamailio box which is currently master will accept that SIP message. This machine can go down at any point -- Kamailio is stateless, so any message at any point in the SIP conversation can come through, and since SIP (is typically) UDP; the protocol already accounts for the fact that a message could be re-transmitted, so even if the Kamailio machine goes down mid-message, the call will still operate.

Each Kamailio box uses the [Kamailio dispatcher](http://kamailio.org/docs/modules/4.3.x/modules/dispatcher.html) and is loaded with the list of Asterisk boxes to dispatch the call too -- and is loaded with a configuration that knows how to detect congestion (so if you're taking a box out of the load-balanced rotation for maintenance, it won't send new calls to it). It then sends the call to the appropriate box.

You'll note on the right, 4 asterisk boxes depicted. The final box being termed "Asterisk@n" -- you can add as many of these Asterisk boxess, and when you add more to the cluster -- using the [kamailio-etcd-dispatcher](https://github.com/dougbtv/kamailio-etcd-dispatcher) -- they're automatically detected and load balanced to, without any further configuration. They're automatically in production and taking calls.

One thing that isn't considered in this project -- is NAT. All the examples are on the same `/24` subnet. I figure that almost everyone's natting concerns are probably different than my own. You'll just need to setup the routing that's most appropriate to your network environment, and setup your Asterisk boxes. This takes out a lot of what could be user-specific configurations and makes for easier testing; giving you the liberty to configure your NAT setup specific to your environment (or lets you avoid NAT entirely, if your environment permits).

As for RTP, each box is aware of it's own Media address, and media is not proxied, but is routed directly to the Asterisk machine from which the call was handled. You may wish to customize this otherwise if you have other needs in your environment.

Asterisk fanboy alert: I haven't yet accounted for this setup using IAX2, but, I know it's possible, and I always prefer to to SIP. If you have a use-case where you can go purely IAX2 I don't want to postpone your joy!

Finally, but possibly most importantly, let's take a look at how the containers and machines inter-operate:

![Deployment scheme](http://i.imgur.com/sOgga6w.png)

There's a lot of detail here, and a lot of connecting pieces. However -- it doesn't require a lot of further configuration (except for the good stuff! Like your dialplans, custom AGI applications, and ancillary applications).

I'll let the diagram do the talking for itself, and we'll get into the details and the guts of it in up-coming articles.

In the next article, we'll get into the nuts and bolts and demonstrate just how easy it is to spin up a new cluster, and we'll demonstrate some of the cool features of this cluster configuration. After we show how it's setup, I'll be coming along with another article which breaks down the components and shows you how to customize the good stuff to suit your production needs.

Thanks for taking the time to read through!
