---
author: dougbtv
comments: true
date: 2025-04-02 11:45:00-05:00
layout: post
slug: cnc-router
title: Sawdust and Serial Ports -- Rescuing a 90s CNC Router
category: nfvpe
---

A couple of summers ago my dad was telling me this story where a customer was in the show room of his sign shop.  A customer's kid, probably prompted by the CRT monitor and yellowed plastic of a late 1990s [eMachines computer](https://en.wikipedia.org/wiki/EMachines) sitting on a desk in the showroom, announces...

> Hey look Mom! It's an antique computer!

![the antique computer, err monitor and keyboard, in all its glory](https://i.imgur.com/BOBfDru.jpeg)

Which I found absolutely hilarious, but... I didn't realize that PC was still alive -- it's a workstation that operates a CNC router that's a core part of my dad's business, [Briar Hill Signworks](https://briarhillsignworks.com/) (don't worry, his website has a 90s feel to it too! But the signs are awesome). He specializes in what I'd consider New England-style carved gold leafed wooden signs. They're gorgeous and he's really perfected the craft, as well his partner who is a painter and (a quite masterful!) gilder, meanwhile he's quite modest about it overall. 

So -- next thought was "OH MY. You are *still running* that same PC to operate the CNC router?" It's a miracle, a damn miracle, that the workstation PC is still running. That eMachines computer has been there for a solid 25 years, dutifully running Windows 98 in order to run some proprietary software, that I later found out, was actually built for Windows 95. I knew it was time to replace that thing, and I told my dad I'd come try to help him when I found some time. Two summers and almost a whole winter passed before I got a fateful call from my dad:

> I can't get files onto the antique computer over the network.

Uh oh, that means production was DOWN at my dad's shop. Which is why I bring to tell you a tale of this retro computing challenge -- quite a departure from my typical tales of open source software, complete with decades old software and hardware, and hardware software license key dongles, and all kinds of other gory stuff.

## The eMachines situation.

It's a got an Intel 266mhz processor running Windows 98 SE, I believe it was originally a sub-$500 machine in maybe 1999. It's got a floppy disk drive, and a CD-ROM drive -- both of which have since failed and both are inoperable. Amazingly, it's got a NIC attached to ethernet and my dad's been sending it files (from another machine, one that he uses more-or-less modern software to design the signs) using Windows File Sharing.

The router is connected over a DB-25 pin cable which is converted to DB-9 pin and works over a serial COM port (or that's how it was last configured, so I kept it that way!). My dad uses two pieces of software on this machine primarily, CASMate and Enroute -- one is like a CAD for sign design and Enroute is used for creating the tool paths (the instructions to send to the CNC router), and then it also has the drivers to actually communicate with the router itself. But you see -- this was rather expensive software at its time, and it's protected with a [software protection dongle](https://en.wikipedia.org/wiki/Software_protection_dongle). I think specifically a HASP (which stands for "Hardware Against Software Piracy") dongle, one that works over a DB25 LPT (parallel printer) port.

![the HASP](https://i.imgur.com/run5G05.jpeg)

*This is the HASP (attached to the new PC workstation), what a relic of a different era.*

My dad's router is a Vytek Rebel 3D. He bought it in 1996, and it was spiffy for its time -- and honestly still really is to this day in a lot of ways (minus the outdated software stack). It's got a 50x50" addressable table with a bit more table outside of that, and the thing is a tank, hell it's nearly 30 now. See -- it's a 3D router. Not 2.5D. With 2.5D, the tool can move in X & Y at the same time, but, Z independently -- with a 3D machine, you can move in all three axis at once. It's critical for making the beautiful v-groove lettering of a carved sign. I thought this might be a software limitation, but nope, it's also apparently a hardware limitation -- there's mechanisms for true Z axis movement, whereas a 2.5D machine can use a solenoid to step along the Z axis.

But, that company doesn't produce CNC routers anymore, they seem to do [CNC laser stuff](https://www.vytek.com/machines) these days, and... It seems non-trivial to replace the head unit on it, which I think I figured out is a Multicam controller.

Growing up, my father started this business in a woodshop that was an outbuilding to our home, and he ran the CNC router in our garage before he moved the business to [a (rather scenic) red barn in Sutton New Hampshire](https://maps.app.goo.gl/4oZ4StV7jDBH2hQy8) (Google street view). As a budding young tech person -- I was *STOKED* for the CNC router to arrive. My next door neighbor was also pretty excited and he RAN to the house to let us know the flatbed delivering the router was pulling up. The CNC router seemed like pure magic. Fascinating to watch it carve a sign. When my dad bought it, it came with a service where someone came to your location to train you how to use it. I begged my dad to let me stay home from school so that I could also learn how to use it. He said no -- if he could go back in time knowing I'll be there to help now, I'll bet he'd say yes, but... Seriously who needs a know-it-all 15 year old (who should probably be at school) bugging you when you're trying to learn something critical for building your business. I still learned a lot along the way (and even worked at the shop for a year, too).

![the barn](https://i.imgur.com/kPrk0C3.png)

My original game plan was to take the HDD out of the eMachines computer and hook it up with an IDE adapter `dd` it and capture an image so that I could run it as a VM. But I explained the risks to my dad -- this computer is 25, and in computer years that's 175, so, any operation -- including taking the hood off to pop out an HDD -- might totally turn this thing to dust. 

He didn't want to do that. Understandably. So, I did it the hard way, a stare-and-compare, and I also was lucky enough to get some files off of it over the network. My dad in desperation had jiggled the ethernet cable attached to it and got the machine running for a few more jobs before I could make it to the barn to help out. Maybe it was just a dodgy network cable that was the problem all along -- who knows, maybe that thing would've gone another 25 years? But, regardless.

My dad was losing sleep over this. I get it. It's production. It's how you make money. I really looked at it as a fun retro computing challenge, and a good way to help family. But, it's family and it's business, and it's production. Guy was freaked, and I don't blame him. This kind of stuff also BOTHERS me. I don't like it when production is down, that bothers me instrinsically. 

## In preparation and the first day on site...

I spec'd out a new school workstation -- I was going to have him buy a laptop, but a buddy of mine recommended that a parallel or serial to USB converter might be too touchy with the precision instruments, so... I looked for a kind of middle-of-the-road workstation and had him get a Lenovo desktop that had a serial and parallel port -- and I also had him buy a smattering of other things at the same time, including PCIe serial and parallel cards, a handful of gender changers for the parallel ports and some peripherals.

The lamest part of the thing is that I opted to have him use VMWare Workstation (which is luckily is a free license these days). Without getting into the details, the gist was that my dad is a windows user, running an old windows for this machine, on a new windows, and we needed the parallel port support -- which is since no longer supported in the latest versions, so I may have advised borrowing an old version from archive.org (I didn't learn that until after the first attempt with the latest version, we settled on a 16.x version).

I figured I’d build out the VM, install the apps, get the dongle working, then try to talk to the router. Easy, right?

Famous last words.

I spent the first day mostly stubbing my toe on the vmware portion, and then guess what I did... Installed old software via CD-ROM drive. Click, bzzt, whirrrrrrr! My dad had saved all the old software and had the CD-ROMs, so, I went through installing it all. I even did it twice because I tried to mess with some windows settings after the first pass, and... I corrupted the O/S haha -- yeah. I forgot to take a snapshot, so, I was pretty sure to take snapshots after that.

I used a bunch of help from ChatGPT, it's great at generating some instructions for what to look for in an old Windoze that I have long LONG forgotten about. Got it hooked up to the file sharing network again, stuff like that. I also ran some troubleshooting scenarios with it.

And... the tour de force of the first day -- I got it to talk to the parallel port dongle and got the proprietary software to load reading whatever auth info from the HASP. I knew a lot of things were going just right to have that happen.

![yup, you can virtualize that](https://i.imgur.com/2yKDLFZ.jpeg)

*Remember that fun visual error? It's like... a failure to clear the video buffer or something? Looks like a win in windows solitaire. I tried to Google the name of this error, there isn't one, apparently*

My dad's partner also had brought some really nice healthy lunch, looked sort of like a bento box with lox and veggies, amazing.

I also got it to talk to the router, kind of. At least... sort of. The router would move to the first position but when it went to drive the tool down the Z-axis, it would go.... excruciatingly slow. I tried a bunch of combinations of different ports, with DB9 adapter and without, onboard parallel and PCIe parallel, that kind of stuff. But wasn't getting much further.

I don't often get to shout "MOVE!" while physically at a workstation, so my dad was subject to many [Nick Burns, Your Company's computer guy](https://www.youtube.com/watch?v=25J3u3P-HHg&list=PLtmabsPPWrkGq2q3veIwG2vJFLdCS6DKB&index=1) jokes throughout the process. I'd try a configuration, then, not remembering all the pieces for what to do, my dad would jump in and start generating toolpaths or sending the job to the router. I noticed that one thing was weird with the job manager, it was like... The process we were using was different, it seemed like it wasn't what my dad was doing. I knew something was different, what was it? It would haunt me for a week.

Have to admit, I was bummed to have to put it down at the end of the day, and felt like I was getting close, but... Until it would route a job -- it wasn't over. And as we know, anything can go wrong, at any time.

I explained the problem to a few friends throughout the course of the project, and a good friend mentioned (paraphrased) "the slow Z movement sure smells like a driver issue." I would keep stewing on that.

But there were things that were retro tech cool that I wound up doing that were successes in their own right, just for the fun of it -- I brought up [Hyperterminal](https://en.wikipedia.org/wiki/HyperACCESS) which I hadn't seen in like, probably 25 years too. I opened up `regedit`, and boy oh boy -- did I hit `windows key + pause/break` dozens of times to look at the device manager.

## The second day, the next week...

I made it down to the shop for a second day. 90 minute drive through the Green Mountains, down into the Connecticut River Valley on a March day, complete with slushy roads and bad coffee from a gas station. Side note: How can those like "ground coffee on demand single serve machines", even Green Mountain Coffee, taste so mediocre? Oh well, I still drank it all morning.

Because one idea that's just a science experiment isn't enough. I also advised him to try the latest edition of Enroute -- which you can buy on a subscription (which is nice compared to the lump sum with no upgrades, as we know from the rest of this story huh). So, we started the day with that. It wasn't just a cake walk, in fact, it was more of a stepping into dog piles kind of walk. Granted, their support was really nice (thanks Jerry!) and we even got to get a laugh out of support when I showed him Enroute 2.1 running on Windows 98. However, they didn't have an immediate solution regarding drivers for our machine, so, by noon time, they'd hit a wall and sent a request to an engineer to get back to us.

Additional benefit of being on the phone with support, a rare opportunity to call my dad "Cheif". No one likes calling their dad "Dad" during a business call, and it's even more awkward to call him by his, you know, actual name. Let's just be honest, just seems weird on both fronts. My own father had worked many years with my grandfather, his father, and he also encountered this as well -- so he called his dad "Cheif". Incidentally, that's also my dad's grandfather name, Cheif. So, I got to call my dad Cheif during the calls.

In parallel while waiting on support, I was also working through the setup of the virtualized old eMachine.

It was time to get to basics: Stare and compare. What's different? Spot the difference between these two pictures. I went step by step through all the stuff I needed to look at. One by one.

![](https://i.imgur.com/Ea1MZaF.jpeg)

*This wasn't exactly what I was comparing, but to give you an idea, it's actually how the "plate" is defined, how you define where the substrate is on the table.*

There was one thing that I knew of and I was seeing, there was a dialogue box that showed the path of the drivers, and it was different on the old machine and on the new VM. One was using `C:\enroute\ODrivers` and the other was using  `C:\enroute\NDrivers` -- I had seen nomenclature during the install for "new drivers" and "old drivers" -- guess what? The old machine was also using the `ODrivers` directory, so, even for the old software, we were using the old drivers.

The thing was... I couldn't change that path. No way to set it. What was I to do?

Well, file contents search for `NDrivers` did the trick -- there was a `C:\windows\enroute.ini` -- and guess what was there? A driver path along with a bunch of other options, including coordinates for where your toolbars go and your whole setup. So I tested changing the driver directory.

My dad sees me editing the `.ini` file in notepad, and he asks:

> Dad: So, you type that, and... it does something?
> Me: I hope so.

It worked to change the displayed driver directory, so I copied the whole `.ini` file over from the old computer. There was actually a paramter named something old `UseOldStyle=true` on the old computer. 

Things started changing QUICKLY. I called my dad over to run a test. 

I knew it was good when the job manager came up to send the job to the CNC router -- this time the process was EXACTLY the way it was when my dad would send a job. He knew the interface immediately -- must've been the `UseOldStyle`, that's what he knew.

We'd be calling back and forth from the show room where the PC workstation is, and the production floor where the router is in the next room over, I'd be yelling "Ready?" and my dad would respond "READY." and we'd send the job. This time though, my dad's instincts kicked in and he removed a checkbox to send the job, he already knew the router was ready.

I watched the progress percentage in the job manager as it sent the job to the router, something like 200KB going over virtualized COM port to the router, in all its 1996 glory.

The router sprung to life. BZZZZT, BRNNNNNNNNNNN, BZZT, BRRRN. Holy smokes.... We got the job to run.

No gonna lie -- I had to hold back a tear. Tears of relief, tears of joy. We got production back online, and at least in a somewhat sustainable fashion that isn't teetering on a budget eMachines PC. My dad even said "I think I could cry." It was an extremely incredible moment with my dad. Seriously a cherished moment. I've had some moments in my tech career where I was so excited that I did cartwheels in an office (apparently I was younger then, I had gotten Asterisk to talk to a Lucent 5E switch!), and even one time I ran out onto the street and yelled with joy! (I found a problem with a backup that was causing a daily outage for weeks in my days as a telco switch tech). But, I can't remember another that quite touched me like that.

Then reality hits: MAKE A VM SNAPSHOT NOW. We had success.

There was a few rough edges, some files would cause a [BSOD](https://en.wikipedia.org/wiki/Blue_screen_of_death) (Wikipedia) when we'd try to import them into Enroute, after me watching a windoze VM bomb out with a BSOD a hundred times, I finally figured out some potion that was "good enough". He also had a way to set a specific `0,0` X,Y coordinates for a home position using CASMate, but I couldn't get CASMate to talk to the router yet, and no obvious way to do it in Enroute, so, we wound up working around it by figuring out a different method for definine "the plate" (as in the above photo).

I kind of figured I'd have to give my dad a whole walkthrough later after getting it all together, but, I think he got the gist of using the VM, the rest of the software running in the VM -- that he knew. But, he didn't need a walk through, I just got [a video of him carving a sign](https://photos.app.goo.gl/jxvYdmP3y7u3zoRX8). I've watched it more times than I'm willing to admit to.