---
author: doug
comments: true
date: 2012-05-18 18:09:12+00:00
layout: post
slug: srtp-asterisk-centos-6-2
title: SRTP + Asterisk + Centos 6.2
wordpress_id: 169
categories:
- Asterisk
---

So, supposedly the libsrtp is included with 1.8.X (at some point) asterisk releases, but... I'm not having luck so, I went through with install it from a tarball and going from there.

What you're going to want to do is, well first, [reference this guy](http://cw.sampas.net/blog/2012/01/asterisk-encryption-gotchas.html). Who got me pointed in the right direction.

You can [download libsrtp straight up](http://srtp.sourceforge.net/download.html) from here. But I was struggling with linkage of the shared object, so [I found this sRPM](ftp://ftp.owlriver.com/pub/local/ORC/srtp/srtp-1.4.4-1orc.src.rpm) (which I found from [this forum post](http://www.pbxinaflash.com/community/index.php?threads/how-to-add-secure-rtp-to-asterisk-1-8.8881/))

Then I had to specify the prefix anyways... I got the code hint from Asterisk ./configure.


    
    
    [user@host]$ ./configure CFLAGS=-fPIC --prefix=/usr
    [user@host]$ make
    [user@host]$ make install
    



Then... recompile Asterisk, starting with a ./configure. Also in "make menuselect" ensure that there's a res_srtp.

<!-- more -->

I went and used [the canonical reference @ wiki.asterisk.org](https://wiki.asterisk.org/wiki/display/AST/Secure+Calling+Tutorial). Which will get you most of the way there. And I bought myself this [Bria softphone for Android](https://play.google.com/store/apps/details?id=com.bria.voip&feature=search_result).

Because, who wants to [have your calls heard from a packet capture](http://wiki.wireshark.org/VoIP_calls)!? By the same token, I am supporting SIP/RTP as well on this machine. "It's your dime" is my motto. 

Here's what my sip.conf looks like afterwards:


    
    
    [general]
    context=default                 ; Default context for incoming calls
    allowoverlap=no                 ; Disable overlap dialing support. (Default is yes)
    udpbindaddr=0.0.0.0             ; IP address to bind UDP listen socket to (0.0.0.0 binds to all)
    tcpenable=no                    ; Enable server for incoming TCP connections (default is no)
    tcpbindaddr=0.0.0.0             ; IP address for TCP server to bind to (0.0.0.0 binds to all interfaces)
    transport=udp                   ; Set the default transports.  The order determines the primary default transport.
    srvlookup=yes                   ; Enable DNS SRV lookups on outbound calls
    externaddr=1.2.3.4
    localnet=192.168.0.0/255.255.0.0
    
    tlsenable=yes
    tlsbindaddr=0.0.0.0
    tlscertfile=/etc/asterisk/keys/asterisk.pem
    tlscafile=/etc/asterisk/keys/ca.crt
    tlscipher=ALL
    tlsclientmethod=tlsv1 ;none of the others seem to work with Blink as the client
    
    [yoursipclient]
    type=peer
    secret=extrasecretpasswordhere 
    host=dynamic
    context=whereyouwantittoland
    dtmfmode=rfc2833
    transport=tls ;# important! (no tls without this)
    mailbox=200
    setvar=MyMail=200
    encryption=yes ;# important! (no srtp without this)
    



Rock out
