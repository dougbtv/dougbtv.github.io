---
author: admin
comments: true
date: 2014-09-06 12:33:40+00:00
layout: post
slug: how-i-learned-to-stop-worrying-and-love-the-firewall-cmd
title: How I learned to stop worrying and love the firewall-cmd
wordpress_id: 333
---

With the advent of Centos 7, I had to face it that `firewalld` is a way of life. I guess it's probably [part of the systemd controversy](http://boycottsystemd.org/). 

I tried to [go back to vanilla iptables](http://stackoverflow.com/questions/24756240/how-can-i-use-iptables-on-centos-7). But... I just felt dirty. I've been living with firewalld on my Fedora workstations for... a while now. But, I never wanted to manage it much. I basically just kept it locked down -- it's workstations anyways, and I was still using iptables on Centos 6. I tried to be lazy -- and run `firewall-config` over an x11 forwarded connection, but... That seemed to be proving harder than actually learning firewall-cmd.

So, I stopped worrying. I might as well use it. 

<!-- more -->

Hell, for about 8 zillion years I've been having to google stuff like "cyberciti iptables drop" to remember what the hell to do with iptables anyways. I just needed a recipe every time. And, then I used firewall-cmd.

Really, I needed to read [the Centos 7 page on using firewalld in detail, before I got it](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Migration_Planning_Guide/sect-Red_Hat_Enterprise_Linux-Migration_Planning_Guide-Security_and_Access_Control.html). 

Once I figured out that I could define what the zones meant by doing an `--add-source`, it clicked for me. So, here's my cheatsheet of what I did to get my bearings, and I have to say, it's kind of a better world. (I'm still struggling with systemctl... I'm like blinded by oldschool sysv style init scripts). I was really trying to just open up for openVPN and then disable SSH was my first goal, so I used two zones "public" (for everything) and then a specific source for the LAN which I called "trusted". Here, I just really play around so I could test it out and proved that it worked according to my assumptions and newly learned tid-bits about firewalld / firewall-cmd


    
    
    # Check out what it looks like...
    
    firewall-cmd --get-active-zones
    firewall-cmd --zone=public --list-all
    
    # Try a port:
    
    firewall-cmd --zone=public --add-port=5060-5061/udp
    firewall-cmd --zone=public --list-ports
    
    # Let's setup the trusted zone:
    
    firewall-cmd --permanent --zone=trusted --add-source=192.168.100.0/24
    firewall-cmd --permanent --zone=trusted --list-sources
    
    # I needed to reload before I saw the changes:
    
    firewall-cmd --reload
    firewall-cmd --get-active-zones
    
    # Now let's configure that up:
    
    firewall-cmd --zone=trusted --add-port=80/tcp --permanent
    firewall-cmd --zone=trusted --add-port=443/tcp --permanent
    firewall-cmd --zone=public --add-port=1194/udp --permanent
    firewall-cmd --zone=public --add-port=1194/tcp --permanent
    
    # Now list what you've got
    
    firewall-cmd --zone=trusted --list-all
    firewall-cmd --zone=public --list-all
    
    
