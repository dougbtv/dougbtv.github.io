---
author: doug
comments: true
date: 2013-09-13 10:07:05+00:00
layout: post
slug: pidora-node-js-i2c-wewt
title: Pidora + Node.js + i2c = Wewt.
wordpress_id: 299
---

So after fighting a lot with this [HiPi perl module](http://raspberrypi.znix.com/hipidocs/installation.htm) (which [unapologetically](https://rt.cpan.org/Public/Bug/Display.html?id=85108) says 'if you don't like that, don't use it'), I decided... Maybe it wasn't worth the long arduous time spent messing with CPAN modules. And, I'm fiddling with Pidora. So far it seems rather convenient for development.

**NOTE! I didn't have luck installing this with Node 0.10.5 -- So I emailed the author of the i2c module. I recommend using 0.10.2.**

So, having an interest in node.js, I found [this i2c npm module](https://npmjs.org/package/i2c)! And all in all it's simple.

[Here's a great over-view on how to get node up and running on your pi](http://blog.rueedlinger.ch/2013/03/raspberry-pi-and-nodejs-basic-setup/)! (It also includes a nice little sys-v style init script, too)

    
    wget http://nodejs.org/dist/v0.10.5/node-v0.10.5-linux-arm-pi.tar.gz
    tar -xzvf node-v0.10.5-linux-arm-pi.tar.gz 
    mkdir /opt/node
    cp -r node-v0.10.5-linux-arm-pi/* /opt/node/


And then add these lines to your **/etc/profile**

    
    ...
    NODE_JS_HOME="/opt/node"
    PATH="$PATH:$NODE_JS_HOME/bin"
    export PATH
    ...


Once that's good to go, log out and back in (or reload your path, if you care to) and then we can install the one RPM we need from YUM and then npm install the module:

    
    yum install gcc-c++
    npm install i2c


I love my perl, but, this module was much less painful to install (including the fact that it took me like 90 minutes to compile the pre-reqs for LWP::Simple just to find out I'd have trouble installing HiPi! Node.js & i2c, here I come!)
