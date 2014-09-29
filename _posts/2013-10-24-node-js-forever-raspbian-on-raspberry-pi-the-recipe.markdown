---
author: doug
comments: true
date: 2013-10-24 12:40:06+00:00
layout: post
slug: node-js-forever-raspbian-on-raspberry-pi-the-recipe
title: 'Node.js + Forever + Raspbian on Raspberry Pi: The Recipe'
wordpress_id: 302
---

Another recipe here for you! Something I thought should be really quick that took me a while. I really like Forever -- it's a great way to get a daemon running with your Node.js project. It requires almost no extra work, and acts as a watchdog to restart your daemon, should it exit prematurely.

Summary: You can simply install Node.js from a binary (look for the [pi version on the node.js distribution downloads](http://nodejs.org/dist/)). You'll copy the binary distribution into /opt/node, and then you'll set your path correctly and install Forever globally. The thing that tripped me up most, was the path. Having the path begin with the node directory is what made me wind up with success.

If you're having this problem where you're running Node.js and Forever and raspberry pi, and you go to run Forever, and you get something like this:

    
    Error: Cannot find module './daemon.v0.6.19'


There's two things going on here, first -- You probably want a newer version of node, which is easy enough to do. Let's go ahead and install it from a binary package from Nodejs.org:

    
    
    sudo mkdir /opt/node
    cd /tmp
    wget http://nodejs.org/dist/v0.10.2/node-v0.10.2-linux-arm-pi.tar.gz
    tar xvzf node-v0.10.2-linux-arm-pi.tar.gz
    sudo cp -r node-v0.10.2-linux-arm-pi/* /opt/node


After you install it, edit your path by modifying /etc/profile


    
    sudo nano /etc/profile



Go ahead and add these two lines before "export $PATH"


    
    NODE_JS_HOME="/opt/node"
    NODE_PATH="/opt/node"
    PATH="$NODE_JS_HOME/bin/:$PATH"



Log out and back in for it to take effect.

But, to install Forever globally, you'll need to do it as root. I couldn't get pure sudo to see the path properly to install, even with "sudo -E", so I did a sudo su, changed my path, and then installed it. Then de-escalated my privileges back to pi (good practice, using sudo su is a stop-gap / work-around here, not a good habit).


    
    
    sudo su
    PATH=/opt/node/bin/:$PATH
    npm install forever -g
    exit
    



Now! You should be able to "run forever" (literally and figuratively, I guess!)


    
    
    # print help text about forever
    forever -h
    
