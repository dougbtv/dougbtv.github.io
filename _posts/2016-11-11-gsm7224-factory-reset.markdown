---
author: dougbtv
comments: true
date: 2016-11-10 14:57:00-05:00
layout: post
slug: gsm7224-factory-reset
title: Factory reset & configure NetGear GSM7224 (without ezconfig)
---

So I am in the process of reconfiguring my home lab setup, which is mostly @ [Laboratory B](https://www.facebook.com/LaboratoryB/) (Burlington Vermont's hackerspace). And I needed a new managed switch, so, I bought an old managed switch (fancy that.) And it's old, and the documentation on it is... quite garbage. It's all there, but, good luck finding what you need in it. So I figured I'd save my steps in case I need to do this again.

First thing for me was that... Well I followed the manual, but, I couldn't get it to listen on the default IP, so I had to console in via serial.


```
[root@whistler doug]# screen /dev/ttyS0 9600
```

And then I logged in (username: admin / password: {blank})

But I tried the `ezconfig` command, what just came back with what looked like a syntax error. Oh well, like it was an unrecognize command.

So I tried to factory reset it by just hitting the "reset" and doing a factory reset at the console, which didn't help me with much. (You get a short chance for it to allow you "start [the] boot menu")

```
[...]
Verifying Operational Code CRC in Flash...100%
Select an option. If no selection in 10 seconds then
operational code will start.

1 - Start operational code.
2 - Start Boot Menu.
Select (1, 2):2



Boot Menu
Options available
 1 - Start operational code
 2 - Change baud rate
 3 - Retrieve event log using XMODEM (64KB)
 4 - Load new operational code using XMODEM
 5 - Run Flash Diagnostics
 6 - Update Boot Code
 7 - Reset the system
 8 - Restore Configuration to factory defaults (delete config files) and reset
 9 - Display operational code vital product data
[Boot Menu] 1

```

Figured out I could show what was there.

```
(GSM7224) #show network

IP Address..................................... 0.0.0.0
Subnet Mask.................................... 0.0.0.0
Default Gateway................................ 0.0.0.0
Burned In MAC Address.......................... 00:0F:B5:FF:17:83
Network Configuration Protocol Current......... DHCP
Management VLAN ID............................. 1
Web Mode....................................... Enable
Java Mode...................................... Enable
```

And then pull up the help for the command to configure, which didn't tell me every one of the parameters, helpful.

```
(GSM7224) #network parms ?

<ipaddr>                 Enter the IP Address.
```

So you actually issue it like so:

```
(GSM7224) #network parms 192.168.1.5 255.255.255.0 192.168.1.1
Network protocol must be none to set ip address.
```

But yeah you gotta unset the network protcol, turns out.

```
(GSM7224) #network protocol none

Changing protocol mode will reset ip configuration.
Are you sure you want to continue? (y/n)y
```
And then issue the command.

```
(GSM7224) #network parms 192.168.1.5 255.255.255.0 192.168.1.1

(GSM7224) #show network

IP Address..................................... 192.168.1.5
Subnet Mask.................................... 255.255.255.0
Default Gateway................................ 192.168.1.1
Burned In MAC Address.......................... 00:0F:B5:FF:17:83
Network Configuration Protocol Current......... None
Management VLAN ID............................. 1
Web Mode....................................... Enable
Java Mode...................................... Enable
```

Voila!