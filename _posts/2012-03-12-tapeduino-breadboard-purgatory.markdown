---
author: doug
comments: true
date: 2012-03-12 17:53:08+00:00
layout: post
slug: tapeduino-breadboard-purgatory
title: 'tapeDuino: breadboard purgatory!'
wordpress_id: 11
categories:
- tapeDuino
---



Then... Out comes the breadboard. And, whoa! It's in a state of disarray. For a while, one of the first things I bumped into was the op amp circuitry in the IC. It took quite a bit of turning, throwing in and tearing out pots, tuning them, trying again, and again and again. My suggestion is to test DTMF output from a variety of devices into the 8870 (the DTMF decoding chip). You'll find a sweet spot that will work for all of the devices. Too much gain, and you'll get nothing. Too little gain, and you'll miss digits. In this process, I was constantly changing up the breadboard, hence the ton of test leads.

In that video you'll see that what I'm decoding is output in binary on 4 LEDs. You'll see the pattern cycling through counting from 0 to 15. (Or if you need a refresher on [counting in binary, wikipedia can help](http://en.wikipedia.org/wiki/Binary_numeral_system#Counting_in_binary))

The [DTMF output example sketch](http://code.google.com/p/arduino-tone/source/browse/trunk/examples/DTMFTest/DTMFTest.pde) from the Tone Library is nice, doubly so in that it [outpulses 867-5309, Jenny](http://en.wikipedia.org/wiki/867-5309/Jenny) (factoid: Most people think [the 555 fictional exchange](http://en.wikipedia.org/wiki/555_(telephone_number)) came about because of that song, but, it only caused a gain in demand for fictional numbers due to it). Is there any other number you should first decode? Historians have unilaterally decided: No.

I've got [some demo code](http://www.pasteall.org/29962/c) from when I was first decoding from the 8870, if it comes in handy.
