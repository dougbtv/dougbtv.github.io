---
author: admin
comments: true
date: 2014-08-02 13:19:29+00:00
layout: post
slug: stegosaurus-js-a-steganographic-device
title: Stegosaurus JS - A steganographic device
wordpress_id: 325
categories:
- Geekery
---

Last weekend I whipped together a [toy steganography device called "Stegosaurus"](https://github.com/LaboratoryB/stegosaurus) [github] -- it will take a PNG image, and using a (very very basic) [steganography](http://en.wikipedia.org/wiki/Steganography) [wikipedia] algorithm stores a payload in the least significant bits of the color definition of pixels in an image. It's a node.js module, and [you can even install it with NPM](https://www.npmjs.org/package/stegosaurus).

Each year a friend of mine puts together a party where people stay for multiple days, and work on art projects at a farm in Vermont. I kept revisiting the wikipedia page over some months, and decided I'd make my own little steganography toy, and just hack it together as quick as possible. Some things need a little work, it's conversion back and forth from binary

It could use a little improvement if anyone is interested in forking it! It needs some testing with binary files. It needs a way to store the length of the message. And ideally, it'd use a pre-shared key (maybe?) to allow you both: A. define where the payload is hidden in the image, and B. actually encrypt the payload (which is, as of now, unencrypted). Which makes it so it doesn't follow [Kerckhoff's Principle](http://en.wikipedia.org/wiki/Kerckhoffs's_principle) [wikipedia].

...Unfortunately every single message is decoded as "[Drink more ovaltine](https://www.youtube.com/watch?v=zdA__2tKoIU)" [youtube] (...just kidding. it'll do whatever message you want)
