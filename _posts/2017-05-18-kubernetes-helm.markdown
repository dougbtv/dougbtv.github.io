---
author: dougbtv
comments: true
date: 2017-05-18 16:30:00-05:00
layout: post
slug: kubernetes-helm
title: Sailing the 7 seas with Kubernetes Helm
category: nfvpe
---

Helm is THE Kubernetes Package Manager. Think of it like a way to `yum` install your applications, but, in Kubernetes. You might ask, "Heck, a deployment will give me most of what I need. Why don't I just create a pod spec?" The thing is that Helm will give you a way to template those specs, and give you a workflow for managing those templates by what it calls "charts". Today we'll go ahead and step through the process of installing Helm, installing a pre-built pod given an existing chart, and then we'll make an existing chart of our own and deploy it -- and we'll change some template values so that it runs in a couple different ways. Ready? Anchors aweigh...

