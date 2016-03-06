---
author: dougbtv
comments: true
date: 2016-03-05 09:30:00-05:00
layout: post
slug: raspberry-pi-version-3-headless-wifi-setup
title: Raspberry Pi Version 3 - Headless Wifi setup
---

So, you got your new Raspberry Pi Version 3 and you want a headless setup, but! You also want wifi, here we go...

Download and load the image onto an SD card, the usual disclaimer that `dd` can screw up your machine reaaaally quick apply. Double check you're using the correct device (my devices used without change, it's `/dev/sdb` here, you might wanna run `lsblk` and check that you're using the right devices)

```
wget --output-document=2016-02-26-raspbian-jessie.zip https://downloads.raspberrypi.org/raspbian_latest
unzip 2016-02-26-raspbian-jessie.zip
# Danger line!
sudo su -c "dd bs=4M if=2016-02-26-raspbian-jessie.img | pv | dd of=/dev/sdb"
```

When done, mount the sd card (I just y'know, take it out of my laptop and plug it back in)

```
[root@talos dougbtv-blog]# lsblk | grep -i sdb
sdb                          8:16   1  29.8G  0 disk 
├─sdb1                       8:17   1    60M  0 part /run/media/doug/boot
└─sdb2                       8:18   1   3.7G  0 part /run/media/doug/443559ba-b80f-4fb6-99d9-ddbcd6138fbd
[root@talos dougbtv-blog]# rpi=/run/media/doug/443559ba-b80f-4fb6-99d9-ddbcd6138fbd
```

Now you can modify the files you'll need, specifically you need to modify `/etc/wpa_supplicant/wpa_supplicant.conf` and add your network info (a couple more hints, [on this stackexchange post](http://raspberrypi.stackexchange.com/questions/10251/prepare-sd-card-for-wifi-on-headless-pi))

If you don't broadcast your SSID, put a `scan_ssid=1` line inside `network={...}`.

```
[root@talos dougbtv-blog]# cat $rpi/etc/network/interfaces | grep -i wpa | tail -n 1
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
[root@talos dougbtv-blog]# 
[root@talos dougbtv-blog]# 
[root@talos dougbtv-blog]# cat $rpi/etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="IamInappropriate"
    psk="fakepassw0rdthatsAQU451goodexamplepassword"
    id_str="Something Descriptive"
}
```

But what if you want a static IP? Go ahead and modify /etc/dhcpcd.conf and add these lines like so smack dab at the bottom. Took a little note from [this stack overflow question](http://raspberrypi.stackexchange.com/questions/37920/how-do-i-set-up-networking-wifi-static-ip).

```
root@raspberrypi:/home/doug# cat /etc/dhcpcd.conf | tail -n 6
# Doug added for static IP.
interface wlan0

static ip_address=192.168.2.180/24
static routers=192.168.2.1
static domain_name_servers=192.168.2.1
```
