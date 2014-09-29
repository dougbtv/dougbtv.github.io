---
author: doug
comments: true
date: 2012-04-22 15:40:52+00:00
layout: post
slug: sparkfun-lab-b
title: Sparkfun @ Lab B!
wordpress_id: 139
---

[![](http://dougbtv.com/wp-content/uploads/2012/04/arduinocompatible-300x225.jpg)](http://dougbtv.com/wp-content/uploads/2012/04/arduinocompatible.jpg)This last wednesday, [Jeff from Sparkfun](http://www.sparkfun.com/static/team) came to [Laboratory B](http://laboratoryb.org/) and held a "train the trainer" workshop... I was lucky enough to participate. Jeff is a pretty chilled out dude, very very casual. The workshop was held at the lab, and the lab opened up for our fellow geeks from [Vermont Makers](http://vermontmakers.org/arduino/sparkfun-electronics-comes-to-vt-and-teaches-our-teachers/).

We got to work making a [Arduino-Compatible PTH Kit](http://www.sparkfun.com/products/10523). Lots of fun. It was nice having my tool box around, I streamlined my process by losing a 16Mhz crystal and luckily I had 'em in my box, plus someone had to rework some caps and they were toasted, but, dun dun dun! 22pF caps to the rescue. And I had a bunch of ATmegas already with bootloaders, so I just popped one of those in. The kit comes with an ATmega328 that's got a nifty little sticker on it with the pinout. S'awesome.

The kit doesn't have a USB-to-RS232 interface, naturally... It's way the most expensive and difficult to deal with piece of an Arduino (at least I only know of SMD parts to do the job [and they're still expensive], I might be looking in all the wrong places). Soooo, yeah, I don't have an FTDI cable, so I'll just program it the way I've been programming it on a perfboard etc for now (e.g. jump Vcc/Vin,RX,TX,Ground & Reset). But now that I look at this way, maybe FTDI is cleaner than my funky little jumpers, I should probably get a FTDI cable.

Speaking of which -- one of the best things out of the whole deal is that I really like the power circuit on the Arduino-compatible that Sparkfun makes. At least, it's better than the simplified version I was using. Of course, that's [open-source hardware, so here's the schematic I like](http://dlnmh9ip6v2uc.cloudfront.net/datasheets/Kits/Arduino-Compatible-PTH-v13.pdf). (In short, they're using a fuse and a diode I wasn't... much cleaner).

The workshop is kind of inspiring me to think about organizing a little series on electronics that I'm interested in. I'm sort of thinking... A retro technology series of sorts. First, a "Make your own synthesizer" (sorta) by making an [Atari Punk Console](http://www.jameco.com/Jameco/PressRoom/punk.html) (most expensive piece is literally the perfboard & a 9v battery). And then maybe a session showing tapeDuino, and combine that with a "learn to solve the Rubik's cube" session. I also picked up a sweeeet piece of retro gear -- but! I'm keeping the cover over it for a minute (while I learn more about it!) -- look for it in a later blog post.

Many thanks to Jeff for holding the workshop! Definitely inspiring!
