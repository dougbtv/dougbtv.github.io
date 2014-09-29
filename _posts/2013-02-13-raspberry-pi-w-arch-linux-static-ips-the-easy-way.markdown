---
author: doug
comments: true
date: 2013-02-13 21:11:28+00:00
layout: post
slug: raspberry-pi-w-arch-linux-static-ips-the-easy-way
title: 'Raspberry Pi w/ Arch Linux: Static IPs the easy way'
wordpress_id: 281
categories:
- raspberryPi
---

Disclaimer! ...Might've been changes to the Arch Distro since I wrote this. I went back to it to try to replicate it, and... No luck. I wound up setting a static DHCP lease on my dd-wrt :(

So you want a static IP in your Arch distribution on your Raspberry Pi because you don't wanna hunt down DHCP huh? Yeah, I don't.

I used [a variety](http://bvanderveen.com/a/rpi-booted-static-ip-ssh/) of tutorials, but.... [Nothing was easy](https://wiki.archlinux.org/index.php/Systemd/Services#Static_Ethernet_network). And almost all of them invariably make it so you have to log in once, first.

I like to change my network settings by pulling the SD card out and then changing the files -- if I'm between locations with my Pi. I might carry it in my backpack and use it on up to 4 networks (home, studio, werk, and [inevitably @ Laboratory B](http://laboratoryb.org/))

Well, take a stroll on easy street. The default configuration lives @ [I think! Chris thanks for pointing out that I had the wrong location in this post!!] /etc/network.d/interfaces/ethernet-eth0

If you can't find the file there, go for a:

    
    find /etc/. | grep -i "eth0"


Find this section and change it up (especially, uncomment it) to suit your network. Also, comment out the DHCP stuff, too (it's up at the top)

    
    ## Change for static
    CONNECTION='ethernet'
    DESCRIPTION='A basic static ethernet connection using iproute'
    INTERFACE='eth0'
    IP='static'
    ADDR='192.168.1.100'
    ##ROUTES=('192.168.0.0/24 via 192.168.1.2')
    GATEWAY='192.168.1.1'
    DNS=('192.168.1.1')


Boot, and enjoy.

Or if you're already at the command line, go ahead and reset it with

    
    [user@host]$ netcfg -r ethernet-eth0
