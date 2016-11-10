---
author: dougbtv
comments: true
date: 2016-11-10 14:57:00-05:00
layout: post
slug: virt-manager-x11
title: virt-manager with X11 forwarding
---

Sometimes you just wanna see your machines in virt-manager, and the console is handy, too, right? `virsh` is awesome and all, but, sometimes I just like having the GUI. For me with a default CentOS 7 minimal install and using oooq (Triple-O Quick start) there's a few extra steps that I needed.

First things first, if you ssh to your box with x11 forwarding like so:

```
ssh -Y stack@eendracht
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

And then `systemctl restart sshd`, and... again

```
ssh -Y stack@eendracht
virt-manager
```

And away you go! But, tip and word to the wise... With virt-manager, it's not going to look for your "user session" by default. So go ahead and in virt-manager GUI go and browse to "File -> Add Connection..." and from there in "Hypervisor" dropdown choose the "QEMU/KVM user session". Now you should see the hosts that oooq created under the stack user.

![virt-manager user session](http://i.imgur.com/OoaNPj4.png "Where to find the user session in virt-manager")

I used to like to just use the virt-manager local to my machine and use an SSH session, however, I haven't yet found a way to directly do that AND also use the user session.