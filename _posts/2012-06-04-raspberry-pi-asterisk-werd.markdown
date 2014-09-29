---
author: doug
comments: true
date: 2012-06-04 20:40:10+00:00
layout: post
slug: raspberry-pi-asterisk-werd
title: Raspberry Pi + Asterisk = werd.
wordpress_id: 181
categories:
- Asterisk
- raspberryPi
---

RaspberryPi is in hand \m/ So what's the first thing I'm going to do with it? Install Asterisk, naturally.

I first fiddled with the [Fedora Remix for Raspberry Pi](http://elinux.org/RPi_Distributions#Fedora_Remix), but, I was finding that I was getting a bunch of errors. Namely, I was getting something like a "long write" warning/failure (which [may be specific to my SD card](http://elinux.org/RPi_VerifiedPeripherals#Working_SD_Cards), I don't know). So I didn't give up, I tried running the [Fedora 17 nightly (with instructions)](http://www.dunkley.me.uk/2012/05/fedora-17-on-raspberry-pi.html) on my RasPi. Welp... Same thing. Seems as though the most favored distro, and the distro (currently) [recommended by raspberrypi.org, is Debian](http://www.raspberrypi.org/downloads).

So, I haven't really been a Debian user. I might've installed it 8 years ago, but, that's the most I ever did. So I'm also loading up with some notes about what was different for me as a mostly CentOS/RHEL/Fedora guy. And of course! My Asterisk build :)

<!-- more -->

I had to figure out [how to manually configure the network](http://wiki.debian.org/NetworkConfiguration#Configuring_the_interface_manually). Your network configs live in /etc/network/interfaces and my config now looks something like

    
    
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet static
            address 192.168.5.222
            netmask 255.255.255.0
            gateway 192.168.5.1
    



ifup and ifdown still apply, as you might guess. Next, the package manager is apt, [and there's a cheat-sheet for that](http://www.cyberciti.biz/tips/linux-debian-package-management-cheat-sheet.html).


    
    
    apt-get install package # Installs a package
    apt-cache search foo    # Searches for "foo" in packages
    



I had to enable ssh daemon as well. I had to generate a rsa keys, then start the service, and enable it at boot (and it's not chkconfig it's update-rc.d). If you want details and a pretty slow rate, but, thank you to the filmer for getting me there, you can check out [this youtube video](http://www.youtube.com/watch?v=SmMMKojOE4U).


    
    
    user@raspberrypi:/dir/$ ssh-keygen
    user@raspberrypi:/dir/$ service ssh start
    user@raspberrypi:/dir/$ update-rc.d ssh defaults
    



Not bad! Everything went swimmingly for me, but... I am short on file system space (I literally had just a couple megs left after an asterisk install [with the tarball deleted] with the default Debian image [which belongs on a 2 gig SD card]). I need to figure out how to expand my partition. Naturally, [eLinux has got the hook-up with a how-to](http://elinux.org/RPi_Easy_SD_Card_Setup#Manually_resizing_the_SD_card_on_Linux). Which I later followed. For the most part, that worked for me. Except the "move" command didn't exactly work. Oh yeah, I did this on another machine (not the Pi) with an SD card reader. The dance I wound up doing looked like:


    
    
    [user@host ~]$ parted /dev/sde
    (parted) unit chs                                                         
    (parted) print                                                            
    Model: Generic STORAGE DEVICE (scsi)
    Disk /dev/sde: 242559,3,31
    Sector size (logical/physical): 512B/512B
    BIOS cylinder,head,sector geometry: 242560,4,32.  Each cylinder is 65.5kB.
    Partition Table: msdos
    
    Number  Start      End         Type     File system     Flags
     1      16,0,0     1215,3,31   primary  fat32           lba
     2      1232,0,0   26671,3,31  primary  ext4
     3      26688,0,0  29743,3,31  primary  linux-swap(v1)
    (parted) rm 2                                                             
    (parted) rm 3                                                             
    (parted) mkpart primary 1232,0,0 239503,3,31                              
    (parted) mkpart primary 239503,0,0 242559,3,31
    # this gave me a couple warnings to which I said "Yup."                       
    (parted) quit                                                             
    [user@host ~]$ mkswap /dev/sde3
    [user@host ~]$ e2fsck -f /dev/sde2
    [user@host ~]$ resize2fs /dev/sde2
    



Quickie tip for screen, while I'm at it. keep a log of your session, hit CTRL+A, then h. To save a "hardcopy" to stop logging to your hardcopy CTRL+A, then m. 

As for installing Asterisk? I'm not the first to go there! [There's raspberry-asterisk.org](http://www.raspberry-asterisk.org/)! Exciting. I would've downloaded his distro, but, there's no seeders on the torrent at the moment. I'll seed it if I can get it. It's great to go through the paces on Debian anyways, good learning experience. 

So I used this guy's notes, and I went ahead with installing just a handful of his dependencies. For example, he's supporting FreePBX so he installs a big tree of dependencies for an http daemon and odbc support, but, I want to slim down to what I use.


    
    
    apt-get -y install make gcc g++ libxml2 libxml2-dev ssh libncurses5 libncursesw5 libncurses5-dev libncursesw5-dev linux-libc-dev openssl libssl-dev
    



Additionally, I was trying to build the certified branch. Buuut... It decided to give up at "src/app.c", where it told me it didn't support arm. Raspberry-asterisk was b[uilding 1.8.13 rc2 (tar.gz)](http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-1.8.13.0-rc2.tar.gz), so, I decided to do the same.

Along with the packages above... I wanted to have TLS/SRTP support, and Jabber/GTalk support, so I added in these modules, which have, at least so far, allowed the correct options to come up in "make menuselect"


    
    
    user@raspberrypi:/dir/$ apt-get install libsrtp0 libsrtp0-dev libgnutls-dev libiksemel-dev
    



And, it works!


    
    
    raspberrypi*CLI> core show version 
    Asterisk 1.8.13.0-rc2 built by root @ raspberrypi on a armv6l running Linux on 2012-06-04 19:26:18 UTC
    raspberrypi*CLI> module show like srtp
    Module                         Description                              Use Count 
    res_srtp.so                    Secure RTP (SRTP)                        0         
    1 modules loaded
    



...Now I gotta backup this image! (and maybe share here, too). Basically, you can reverse the dd command to flash the image in order to back it up. But, in my case, I had a 2gig image on a 16gig card. So I wound up with a monstrous 16 gig image. Naturally, when it gzipped, it became tiny. However, it was cumbersome, so I wanted to cinch it up. I decided to just limit the dd run with a command like:


    
    
    [user@host dir]$ dd bs=1M count=2000 if=/dev/sde of=debian_asterisk_compact.img
    
