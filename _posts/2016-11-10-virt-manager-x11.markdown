---
author: dougbtv
comments: true
date: 2016-10-27 09:58:00-05:00
layout: post
slug: virt-manager-x11
title: virt-manager with X11 forwarding
---

So, usually I am like the guy to poo-poo using a GUI for something. Like, I've been using MySQL for over 15 years, I hate using a GUI for it, it just slows me down. But, everyone becomes a newb at something at some point, so, a GUI can help you out. It can introduce you to new ideas, and it can help guide you to things that you may have been unaware of, and it can be pragmatic -- it can help you do something right now that you couldn't do without (because you're new, and you don't know yet)

So I use libvirt for VMs for development on my laptop for a long time now. And yes, I'd create XML definitions and I'd manage them with Ansible, and do fairly advanced things, but, I've got to use them at a deeper level now, and I'm feeling like a n00b. 

First things first, so you ssh to your box:

```
ssh -X root@eendracht
```

But, I'd get a 

```
X11 forwarding request failed on channel 0
```

So I found out that I needed to change `/etc/ssh/sshd_config` specifically I [needed to have this line](http://www.cyberciti.biz/faq/how-to-fix-x11-forwarding-request-failed-on-channel-0/):

```
X11UseLocalhost no
```

And I also needed to install `xauth`

```
sudo yum install -y xauth
```

And then did a `systemctl restart sshd`, and that went away, but, I'd go to start virt-manager, and I'd get:

```
[root@eendracht ~]# virt-manager 
[root@eendracht ~]# 
** (virt-manager:28203): WARNING **: Could not open X display

(virt-manager:28203): Gtk-CRITICAL **: gtk_settings_get_for_screen: assertion 'GDK_IS_SCREEN (screen)' failed
(virt-manager:28203): Gtk-CRITICAL **: _gtk_settings_get_style_cascade: assertion 'GTK_IS_SETTINGS (settings)' failed
(virt-manager:28203): Gtk-CRITICAL **: _gtk_style_provider_private_lookup: assertion 'GTK_IS_STYLE_PROVIDER_PRIVATE (provider)' failed
(virt-manager:28203): Gtk-CRITICAL **: _gtk_css_lookup_resolve: assertion 'GTK_IS_STYLE_PROVIDER_PRIVATE (provider)' failed

[... and a lot more ...]
```

But man all of this is a pain when you can just have `virt-manager` on your client machine and do "File -> New Connection" and use SSH and provide the hostname and user. And you already have SSH keys. That's the way to do it.