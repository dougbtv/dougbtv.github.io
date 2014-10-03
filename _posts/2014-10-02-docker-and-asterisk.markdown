---
author: dougbtv
comments: true
date: 2014-10-02 12:33:40+00:00
layout: post
slug: docker-and-asterisk
title: Docker and Asterisk
wordpress_id: 333
---

Let's get straight to the goods, then we'll examine my methodology.

You can clone or fork my [docker-asterisk project on GitHub](https://github.com/dougbtv/docker-asterisk). And/or you can [pull the image from dockerhub](https://registry.hub.docker.com/u/dougbtv/asterisk/).

Which is as simple as running:

    docker pull dougbtv/asterisk

Let's inspect the important files in the clone

    .
    |-- Dockerfile
    |-- extensions.conf
    |-- iax.conf
    |-- modules.conf
    `-- tools
        |-- asterisk-cli.sh
        |-- clean.sh
        `-- run.sh

In the root dir:

* `Dockerfile` what makes the dockerhub image `dougbtv/asterisk`
* `extensions.conf` a very simple dialplan
* `iax.conf` a sample iax.conf which sets up an IAX2 client (for testing, really)
* `modules.conf` currently unused, but an example for overriding the modules.conf from the sample files.

In the `tools/` dir are some utilities I find myself using over and over:

* `asterisk-cli.sh` runs the `nsenter` command (note: image name must contain "asterisk" for it to detect it, easy enough to modify to fit your needs)
* `clean.sh` kills all containers, and removes them.
* `run.sh` a suggested way to run the Docker container.

That's about it, for now!

---

There's a couple key steps to getting Asterisk and Docker playing together nicely, and I have a few requirements:

* I need to access the Asterisk CLI
* I also need to allow wide ranges of UDP ports.

On the first bullet point, we'll get over this by using `nsenter`, which requires root or sudo privileges, but, will let you connect to the CLI, which is what I'm after. I was inspired to use this solution from this [article on the docker blog](http://blog.docker.com/2014/06/why-you-dont-need-to-run-sshd-in-docker/). And I got my method of running it [from coderwall](https://coderwall.com/p/xwbraq).

On the second point... Docker doesn't like UDP it seems (which is what the VoIP world runs on). At least in my tests trying to get some IAX2 VoIP over it, on port 4569. (It's an easier test to mock up than SIP!)

So, I settled on opening up the network to Docker using the `--net host` parameter on a `docker run`.

At first, I tried out bridged networking. And maybe not all is lost. Here's the basics on bridging [here @ redhat](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/s2-networkscripts-interfaces_network-bridge.html) I followed. Make sure you have bridge-utils package: `yum install -y bridge-utils`. But, I didn't have mileage with it. Somehow I set it up, and it borked my docker images from even getting on the net. I should maybe read the [Docker advanced networking docs](https://docs.docker.com/articles/networking/#building-your-own-bridge) in more detail. Aaaargh. 

Some things I have yet to do are:

* Setup a secondary container for running FastAGI with xinetd.

I'm thinking I'll running xinetd in it's own container and connect the asterisk image with the xinetd image for running FastAGI.