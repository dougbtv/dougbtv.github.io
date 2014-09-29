---
author: doug
comments: true
date: 2012-05-29 14:25:41+00:00
layout: post
slug: asterisk-intermittent-rtp-issues-on-natd-clients
title: 'Asterisk: Intermittent RTP issues on NAT''d clients?'
wordpress_id: 177
categories:
- Asterisk
---

Or any clients for that matter.

I spent what would've been hours of joy playing and playing with the fun toys of Asterisk, but, I found myself trouble shooting intermittent problems with clients behind a NAT. What threw me off was that one softphone (Bria) would send/receive RTP every, single, time. But others, not so much.

What was it?

Turned out that I had the range "10000-10200" in my rtp.conf.

But in my firewall? I had "10100-10200".

It was a typo in my rtp.conf. Dammnit. So hopefully if you are googling for intermittent RTP problems with natted clients, you find this and you double check it. Man was that frustrating!!!
