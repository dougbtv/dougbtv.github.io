---
author: doug
comments: true
date: 2012-03-26 01:31:32+00:00
layout: post
slug: ack-post-this-on-mythtv-wiki-was-down-when-i-tried
title: Ack, post this on mythtv wiki (was down when I tried)
wordpress_id: 112
---

===Fedora 16: Turn off "screen blanking"===

Upstream Gnome has decided to not use a screensaver anymore, and just uses a technique to blank the screen, and turn off your monitor. This may not be desirable with your MythTV front-end in order to disable it, you may need to disable DPMS (Energy Star) at start-up and

* Create a startup script in:
** /etc/X11/xinit/xinitrc.d/xset-dpms.sh
** containing:
#!/bin/bash
xset -dpms

* Make it executable:
chmod a+x /etc/X11/xinit/xinitrc.d/xset-dpms.sh

* Install dconf-editor:
yum install dconf-editor

* From a terminal in Gnome run
dconf-editor
** From the dconf-editor GUI look in: org->gnome->desktop->screensaver.
** Set the "idle-activation-enabled" boolean to false (unchecked).

* Grin with a disabled screen-saver / screen-blanking system.

(Credit to [http://forums.fedoraforum.org/showpost.php?p=1547438&postcount=16 rabrant on fedoraforum.org])
