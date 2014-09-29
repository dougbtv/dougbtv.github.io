---
author: admin
comments: true
date: 2014-08-27 20:02:19+00:00
layout: post
slug: oh-t3h-noes-you-just-borked-your-fedora-20-install
title: Oh t3h noes! You just borked your Fedora 20 install!
wordpress_id: 329
---

So, I made a nice boo-boo with my Fedora 20 install on my laptop this morning. I accidentally rebooted before a yum update was finished. Annnd.... I figured it was better off to re-install rather than try to figure out how to recover -- doubly so since I lost network connectivity, making it really hard to get references.

If you're a MEAN stack developer on Fedora 20, you might install some of the same tools I do when I reinstall, so I took some note for me, and... For you.

First things first, install chrome: install chrome: http://www.if-not-true-then-false.com/2010/install-google-chrome-with-yum-on-fedora-red-hat-rhel/ & then I disable SELinux.


    
    
    # Command line basic tools.
    yum install nano terminator rsyslog
    
    # install your MEAN stack developers stuff
    yum install mongodb mongodb-server nodejs nginx npm git rubygem-compass
    
    # install robomongo
    # http://robomongo.org/
    yum install glibc.i686 libstdc++.i686
    rpm -ivh robomongo_version.rpm
    
    # install your super sweet grunt & yeoman globally install npm packages
    npm install -g grunt-cli yo
    
