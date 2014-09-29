---
author: doug
comments: true
date: 2012-06-05 18:06:30+00:00
layout: post
slug: asteriskdebian-raspberrypi-image-for-download
title: Asterisk+Debian RaspberryPi Image for Download
wordpress_id: 208
categories:
- Asterisk
- raspberryPi
---

Torrent: [debian_asterisk18_2012-06-04.img.gz.torrent](http://dougbtv.com/debian_asterisk18_2012-06-04.img.gz.torrent) [[Magnet URL](magnet:?xt=urn:sha1:c21807659d045a3b9e942313d516412072db1ac5&dn=debian_asterisk18_2012-06-04.img.gz﻿)]
SHA-1: c21807659d045a3b9e942313d516412072db1ac5

Debian Squeeze (debian6-19-04-2012) + Asterisk 1.8.13rc2 For Raspberry Pi. The Image.

Asterisk features installed:
- TLS / SRTP
- Jabber / GTalk (for Google Voice)
- No ODBC.
- No FreePBX.

The only alterations are:
- Root password is: root
- User "pi" password is: raspberry (that's actually default)
- Static IP @ 192.168.1.101
- SSH Daemon starts at boot
- Dependencies for Asterisk installed
- Asterisk 1.8.13 rc 2 installed.
- Asterisk source in /usr/src
- Asterisk starts at boot

Requires: 2 gig (preferably 4 gig or more) SD card.

You should gunzip the image (I've made that mistake!).
You must resize the partition after you flash it. 
For instructions see: [http://elinux.org/RPi_Easy_SD_Card_Setup#Manually_resizing_the_SD_card_on_Linux](http://elinux.org/RPi_Easy_SD_Card_Setup#Manually_resizing_the_SD_card_on_Linux)

Flashing the card is easy, too: [http://elinux.org/RPi_Easy_SD_Card_Setup](http://elinux.org/RPi_Easy_SD_Card_Setup)

Want to know more? [Here's where I documented my steps while creating this image](http://dougbtv.com/?p=181).
