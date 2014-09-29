---
author: doug
comments: true
date: 2012-03-15 15:13:49+00:00
layout: post
slug: tapeduino-fritzing-diagrams
title: 'tapeDuino: Fritzing Diagrams'
wordpress_id: 81
categories:
- tapeDuino
---

[![tapeDuino: Fritzing Breadboard](http://dougbtv.com/wp-content/uploads/2012/03/td-fritzing-breadboard-300x247.png)](http://dougbtv.com/wp-content/uploads/2012/03/td-fritzing-breadboard.png)[ Check out the project page on Fritizing for tapeDuino](http://fritzing.org/projects/arduino-dtmf-encoderdecoder-tapeduino/). I've got a Fritzing files and some example code for writing DTMF tones, and reading DTMF tones using the MT8870.Included is the real meat of the circuitry for tapeDuino. A lot of bells and whistles isn't provided. But, it gives you what you need in order to start using an MT8870 DTMF decoder IC, and start seeing some info from it.
A few things to consider:

1. The schematics use an Arduino Uno. This is great for testing, however, in application I needed to use a Mega. You use up all the pins on the Uno quickly. When you attach an ethernet shield, you lose a number of pins as well. And, the Tone library uses 2 out of 3 of your timers when you play 2 tones at once (the "dual" in Dual Tone Multi-Frequency). The mega has more timers.

2. The LEDs are completely optional, but, very nice when you're trying to debug. In my final circuit, I kept them -- but! I used a jumper to turn them on or off. So I could look at them while debugging, but, not have something needlessly flashing on the inside of my project box while it's closed up, etc.[![tapeDuino: fritzing schematic](http://dougbtv.com/wp-content/uploads/2012/03/td-fritizing-scheme-300x133.png)](http://dougbtv.com/wp-content/uploads/2012/03/td-fritizing-scheme.png)

3. In my final application, I've gone totally mono. However, I still use the 3.5mm headphone jack, and I only use one. I use_ Red - Right - READ_, and then _White - WRITE _(and therefore, left). These mnemonics help me out, cause I go from 3.5mm stereo to RCA on the tape deck. It took me a long while to realize that I could do stereo, and therefore two channel recording. I could potentially double my data rate (!), but it'd require two MT8870s (which aren't expensive), and a lot of noodling on the code. I'm happy with mono in my v1.



Here's some tips to get you moving in the right direction right off the bat. Once you've got the MT8870 circuit setup. There's no need to initially connect it to the Arduino. Just power the circuit (through your bench power supply, or your Arduino, or whatever you prefer). Next... The big thing to pay attention to is the two pots that are between the 3.5mm jack and the 8870. These are for your 8870's op-amp circuit. Connect a 3.5mm stereo audio cable from your computer, or your phone, and play DTMF tones. The iphone plays the tones ULTRA quiet from the headphone jack, and they do some kind of voodoo bullshit (pardon my french) with their electronics -- so don't trust it as far as you can throw it (you could huck one really good). What I would do is use google voice to call my own cell phone and send aux audio output to the DTMF circuit from my computer. Then I'd hit DTMF tones from either google voice or my phone and see if it registered on the LEDs. If it doesn't? You'll need to adjust the pots. Keep adjusting then hitting tones.

For what worked for me? For the left-most pot (labeled R6 in the schematic) I had it set at 25k & the right most post (labeled R7 in the schematic) set at 75k. That worked for every device I had to throw at it.

If you read the 8870 datasheet, you'll see that this basically matches their exact example circuit. I questioned it a number of times, to no avail. That example circuit is great.

I've connected the "Delayed Steering" pin to Pin 2 of the Arduino. This is so you can use it as an interrupt if need be. My final implementation does exactly this. It's a lot better than constantly polling for this pin.

What does the "Delayed Steering" pin do? When a tone is successfully decoded, this pin goes voltage high. That lets you know you can read from the Q0-Q3 pins -- which represent the nibble of data that is each of the 16 DTMF tones.

The most complex part of the whole thing, is actually the Guard Time Adjustment circuitry (which is on pins 16,17,18 of the 8870). This part of the circuit allows you to customize the amount of time it takes to decode a DTMF tone. I might make an article simply about this part of the circuit.

Let's just put it this way -- in order for me to REALLY figure out how to adjust my guard time, I had to go to my buddy LordPants (name changed to protect the innocent!) who's also a goddamn rocket scientist. And I had him read the data sheet. With one super smart dude (him) and one ultra stubborn dude (me), we finally got it figured out.

But, I'll put it this way: Use my schematic, and what you'll be adjusted for is this: If you play a DTMF tone for 20ms, and then NOT play a DTMF tone for 10ms, you'll be inside the specs of the MT8870-DE's abilities. I suffered with this one for a long time, and it took me a lot of read/write testing to finally narrow down on it.

I've got spreadsheets and more documentation on the adjustments of this, and even with that... I still screwed up. But, I may just post it in case someone like me goes googling around for it, and finds out... They've basically got to tap the shoulder of their smartest friend in order to figure it out.

Hopefully, this gets someone up and running! *thumbs up*




