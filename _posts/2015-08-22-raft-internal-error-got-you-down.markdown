---
author: dougbtv
comments: true
date: 2015-08-22 09:08:00-05:00
layout: post
slug: raft-internal-error-got-you-down
title: etcd "raft internal error" got you down? 
---

So you're rolling along with CoreOS and then you go and make your cluster rediscover itself after some configuration changes, and you the cluster is broken, and no one's discovering anything, and etcd processes aren't starting, and you're ready to pull your hair out because get something in your etcd logs that looks like:

```
fail on join request: Raft Internal Error 
```

I think I've got the fix -- clear our the caching that etcd has, and go ahead and feed the cluster a new discovery token / discovery URL.

Ugh! It had been killing me. I'd been running CoreOS with libvirt/qemu, and I had everything lined up to work, and it'd usually work well the first time I booted the cluster up, but... I'd go and reboot the cluster, and feed it a new discovery URL, and... bam... I'd start getting these left and right with etcd 0.4.6 & 0.4.9.

I'd keep [running into this github issue](https://github.com/coreos/etcd/issues/863), which shows that I'm not the first to have the problem, but, no solution? The best I could find in other posts was to upgrade to etcd2 (which you do by just running the service called `etcd2` instead of just plain `etcd`). I gave it a try, but, I wasn't having much luck, and I was already deep down the path of figuring this out.

So, I slowed myself down, and I started building from scratch, and slowly watching the machines come up, one by one. Then, I tried powering down, reloading cloud configs, and watching them one by one... I noticed, that the first machine to come up, on it's own... with a brand new discovery token... Would remember -- that it had 4 friends. What!? You're not supposed to know yet! 

I don't claim to grok the internals, but, what I did understand is that it was remembering something, that in my opinion it shouldn't have. How might I clear it's memory and work like the first time I started up the cluster?

That being said, I made myself a bit of an experiment, and I went and did a:

```
systemctl stop etcd
rm /var/lib/etcd/log
rm /var/lib/etcd/conf
shutdown now
```

...I personally put this in an ansible script, and use it during a playbook that I call "cluster rediscovery" -- which is pretty nice when you're say, on a laptop and bringing the cluster up basically every time you open up your laptop.