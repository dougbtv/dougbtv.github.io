---
author: dougbtv
comments: true
date: 2014-12-15 16:10:00+00:00
layout: post
slug: introducing-bowline-io
title: Introducing Bowline.io
---

I'm introducing my new tool [Bowline.io](https://bowline.io) -- a tool to tie your Docker images to a build process. It has a bunch of features that I'm rather proud of:

* It automatically builds images based on git hooks (or polling over HTTP)
* It logs the results of those builds, so you can see the history.
* It'll allow you to build unit tests into your Dockerfiles so you know if it's a passing build
* It's a front-end to a Docker registry, to give you a graphical view of it
* It's an authenticated Docker registry, which users can manage their creds online.
* It keeps some meta data about your "knot", like a `README.md` and it syntax highlights your `Dockerfile`
* It's compatible with docker at the CLI, like `docker login` and therefore `docker push` and `docker pull`

And what it really boils down to is that it's kind of a Dockerhub clone that you can run on your own hardware. Sort of how you might use Dockerhub Enterprise, but, it's open source. You can just spin it up and use it to your heart's content. Check out [the documentation on how to run it locally](https://github.com/dougbtv/bowline/blob/master/docs/RunningLocally.md) for yourself -- you won't be surprised to find out that it runs in Docker itself.

It's what happened to my "asterisk autobuilder for docker" I wrote about in a previous post. That tool was working out rather well for me, and when I went to extend what was previously an IRC bot to handle multiple builds -- I realized that I could extend it to a web app with some ease (It's a NodeJS application, and already was starting to expose an API). 

I've got even more plans for it in the future, some of which are:

* Continuous deployment features (make an action after your build finishes)
* More explicit unit testing functionality
* The capability to test an "ecosystem" of Docker containers together (like a webapp, a db & a web server for example)
* Ability to create geard & fleet unit files for deployment
* A distributed build server architecture so you can run it on many boxen (it's been planned in the code, but, not yet implemented)

Feel free to give it a try @ [Bowline.io](https://bowline.io) -- I'm looking forward to having more guinea pigs check it out, use the hosted version @ Bowline.io, or running it on their own boxen -- or best yet! Contributing a PR or two against it.
