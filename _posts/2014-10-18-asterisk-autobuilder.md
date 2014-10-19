---
author: dougbtv
comments: true
date: 2014-10-18 12:33:40+00:00
layout: post
slug: asterisk-autobuilder
title: Asterisk Autobuilder for Docker
---

I've gone ahead and expanded upon my [Asterisk Docker image](https://registry.hub.docker.com/u/dougbtv/asterisk/), to make a system that automatically builds a new image for it when it finds a new tarball available.

Here's the key features I was looking for:

* Build the Asterisk docker image and make it available shortly after a release
* Monitor the progress of the build process
* Update the Asterisk-Docker git repo

To address the secondary bullet point, I made a REPL interface that's accessible via IRC -- and like any well behaved IRC netizen; it posts logs to a pastebin.

Speaking of such! In the process, I made an [NPM modules for pasteall](https://www.npmjs.org/package/pasteall). If you don't know [pasteall.org](http://www.pasteall.org) -- it's the best pastebin you'll ever use.

You can visit the bot in `##asterisk-autobuilder` on [freenode.net](https://freenode.net/).

As for the last bullet point, when it finds a new tarball it dutifully updates the asterisk-docker github repo, and makes a pull request. [Check out the first successful one here](https://github.com/dougbtv/docker-asterisk/pull/16). You'll note that it keeps [a link to the pasteall.org logs](http://www.pasteall.org/54631/text), so you can see the results of the build -- in all their gory detail, every step of the docker build.

I have bigger plans for this, but, some of the shorter-term ones are:

* Allow multiple branches / multiple builds of Asterisk (Hopefully before Asterisk 13!!)
