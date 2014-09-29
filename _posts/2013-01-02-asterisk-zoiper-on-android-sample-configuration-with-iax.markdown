---
author: doug
comments: true
date: 2013-01-02 18:28:42+00:00
layout: post
slug: asterisk-zoiper-on-android-sample-configuration-with-iax
title: Asterisk + Zoiper on Android Sample Configuration, with IAX
wordpress_id: 240
categories:
- Asterisk
---

Yep, this is really the thing that makes Zoiper worthwhile as a softphone is that you can use an IAX connection. Pretty nice. 

I figured I'd post up my configuration in case any one else is looking for an example:

Here's my /etc/asterisk/iax.conf


    
    
    [general]
    bandwidth=low
    disallow=lpc10                  ; Icky sound quality...  Mr. Roboto.
    jitterbuffer=no
    forcejitterbuffer=no
    autokill=yes
    
    [theuser]
    type=friend
    auth=md5
    context=yourextensioncontext
    trunk=no
    secret=makeagoodsecret
    defaultuser=theuser
    username=theuser
    host=dynamic
    encryption=yes
    requirecalltoken=no
    disallow=all
    allow=gsm
    



And on my Zoiper softphone:

Account name: [arbitrary]
Host: [your hostname/ip/etc]
Username: theuser
Password: makeagoodsecret
Context: yourextensioncontext

I whiffed a good couple times trying to get it to work, it seemed to be a combo between A. not setting the context in Zoiper, and B. possibly having the wrong "type" (friend/peer/user) in the iax.conf section.
