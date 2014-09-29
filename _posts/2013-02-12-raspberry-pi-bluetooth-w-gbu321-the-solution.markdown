---
author: doug
comments: true
date: 2013-02-12 21:25:37+00:00
layout: post
slug: raspberry-pi-bluetooth-w-gbu321-the-solution
title: Raspberry Pi Bluetooth w/ GBU321 -- the solution.
wordpress_id: 245
categories:
- raspberryPi
---

Got yourself one of these? A GBU321, as recommended in the [verified peripherals](http://elinux.org/RPi_VerifiedPeripherals#USB_Bluetooth_adapters)? 

I didn't have a cake walk. Namely, I kept running into an error like "hci0 command tx timeout" -- and lots of flakiness. Sometimes my hci0 would come up in a up state, sometimes a down. Here's my lsusb:


    
    
    pi@raspberrypi ~ $ lsusb
    [...]
    Bus 001 Device 004: ID 0a5c:4500 Broadcom Corp. BCM2046B1 USB 2.0 Hub (part of BCM2046 Bluetooth)
    Bus 001 Device 005: ID 0a5c:4502 Broadcom Corp. Keyboard (Boot Interface Subclass)
    Bus 001 Device 006: ID 0a5c:4503 Broadcom Corp. Mouse (Boot Interface Subclass)
    Bus 001 Device 007: ID 0a5c:2148 Broadcom Corp. BCM92046DG-CL1ROM Bluetooth 2.1 Adapter
    



I have what I believe to be the silver bullet. It appears that [the Raspberry Pi Github has an issue listed](https://github.com/raspberrypi/linux/issues/66) that is what did the trick, for mine.

It appears that the issue might be with the USB speed negotiation. So, here's a work-around in your /boot/cmdline.text, add "dwc_otg.speed=1" near the end. My entire line now looks like:


    
    
    dwc_otg.lpm_enable=0 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline dwc_otg.speed=1 rootwait
    



To install up the requirements -- I followed [this method for a PS3 controller](http://forum.stmlabs.com/showthread.php?tid=4273) as recommended by[ a guy in this raspi forum post (which I mean to comment on: note to self!)](http://www.raspberrypi.org/phpBB3/viewtopic.php?f=28&t=9585)


    
    
    /* asdf foo */
    root@raspberrypi:~# apt-get install bluetooth
    root@raspberrypi:~# apt-get install bluetooth --fix-missing
    root@raspberrypi:~# reboot
    root@raspberrypi:~# hciconfig -a
    root@raspberrypi:~# hciconfig hci0 piscan
    root@raspberrypi:~# bluez-simple-agent 
    Agent registered
    RequestConfirmation (/org/bluez/1824/hci0/dev_B0_D0_9C_58_XX_YY, 614348)
    Confirm passkey (yes/no): yes
    root@raspberrypi:~# bluez-test-device trusted B0:D0:9C:58:XX:YY yes
    
    
