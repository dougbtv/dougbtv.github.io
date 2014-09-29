---
author: doug
comments: true
date: 2012-03-10 00:52:18+00:00
layout: post
slug: hello-world
title: tapeDuino -- the concept.
wordpress_id: 1
categories:
- tapeDuino
---

[![](http://dougbtv.com/wp-content/uploads/2012/03/schematic-tapeDuino-Integrated-v2_bb-300x200.png)](http://dougbtv.com/wp-content/uploads/2012/03/schematic-tapeDuino-Integrated-v2_bb.png)Over a course of visits to the studio space I share... I kept seeing an old school cassette walkman, and a bunch of (bad) old tapes on the desk of one of my space-mates. After some time passed, I thought he was maybe building a [datasette](http://en.wikipedia.org/wiki/Commodore_Datasette)[ clone. ](http://dougbtv.com/wp-content/uploads/2012/03/schematic-tapeDuino-Integrated-v2_bb.png) I loved the C64 Datasette! I was a bit too young to own my own, however I was envious of my cousin's datasette, man, too cool.

That, ummm, wasn't the case at all. In fact, I'm still not sure why he's got the tapes and the walkman. That being the case... I decided, dun, dun dun! I would make my own datasette clone. Using an arduino as the brains to back it. About 25 years later, I could have my own datasette \m/



What tapeDuino really is; is a Casette Tape NAS, a home-brew NAS. Or if I want to explain it to my mom "It's kinda like a harddrive, or a memory stick, however, when it stores data on a cassette tape, and that's supposed to be kinda funny. And it's homemade." Or if my wife were to explain it she'd probably call it "my girlfriend" or "a rube goldberg contraption". And yes... It is intentionally silly; come on, a tape deck for media. The tapeDuino unit sits independent of a host, and connects to the ethernet; with a client which runs on (maybe) any *nix host with a Perl implementation.

<!-- more -->

One of the first things I was thinking of was; how am I going to store the data as tones on the tape deck?

After some noodling about it, and being kind of a telephony weirdo myself; I decided I would use [DTMF](http://en.wikipedia.org/wiki/Dual-tone_multi-frequency_signaling) to store the tones onto tape. The reasons I picked DTMF are...



	
  * It's an easy existing way of storing a value with a tone that already has a spec.

	
  * I could use an IC to offload most of the read operation to; specifically the MT8870 [[datasheet](http://www.wulfden.org/downloads/datasheets/M-8870-01.pdf)] [[second datasheet]( http://www.zarlink.com/zarlink/mt8870d-datasheet-oct2006.pdf )]

	
  * The DTMF spec calls for 16 symbols. Conveniently, that's hexadecimal. So each symbol is a [nibble](http://en.wikipedia.org/wiki/Nibble) (4 bits) of data.

	
  * It's kinda retro, just like a tape deck.


So I started to do some homework. I found that [RazorConcepts had designed a PCB ]( http://razorconcepts.net/dtmf.html)to convert the 4 bit output of a MT8870 to decimal, which [you can buy on BatchPCB]( http://batchpcb.com/index.php/Products/27676). He also [discussed it on a forum]( http://forum.allaboutcircuits.com/showthread.php?t=30775) before finalizing, which I found interesting. There's a also a guy who made [an Arduino shield]( http://w2mh.wordpress.com/2010/06/03/dtmf-shield/) based on the RazorConcept's design. And this guy [posted on the Arduino.cc forum]( http://www.arduino.cc/cgi-bin/yabb2/YaBB.pl?num=1246431289) about DTMF circuitry with an 8870. And another post from that forum gave me the idea to put [LEDs on the 8870's output pins](http://www.arduino.cc/cgi-bin/yabb2/YaBB.pl?num=1284828366), for debug.

As for output, I do have an IC I bought from Newark, which is a DTMF generator. It's a [Holtek HT9200A](http://www.newark.com/jsp/search/productdetail.jsp?SKU=53M7848) [[datasheet](http://www.farnell.com/datasheets/79214.pdf)]. However, it's a surface mount IC, argh. Which kinda turned me off. So I went ahead with using the [Arduino Tone Library](http://code.google.com/p/rogue-code/wiki/ToneLibraryDocumentation). Which conveniently has a [DTMF example sketch](http://code.google.com/p/arduino-tone/source/browse/trunk/examples/DTMFTest/DTMFTest.pde) (however, you'll have to go to wikipedia to get the missing 4 tones from the spec).

Given what I had found, I was ready to whip out the breadboard, and start building out the discrete elements.
