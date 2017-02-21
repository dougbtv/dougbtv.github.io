---
author: dougbtv
comments: true
date: 2017-02-09 15:10:01-05:00
layout: post
slug: multus-cni
title: So you want to expose a pod to multiple network interfaces? Enter Multus-CNI
category: nfvpe
---

Sometimes, one isn't enough. Especially when you've got network requirements that aren't just "your plain old HTTP API". In the telephony world, something we love to do is isolate our signalling, media, and management networks. If you've got those in separate NICs on your container host, how do you expose them to a Kubernetes pod? Let's plug in the [CNI](https://github.com/containernetworking/cni) (container network interface) plugin called [multus-cni](https://github.com/Intel-Corp/multus-cni) into our Kubernetes cluster and we'll expose multiple network interfaces to a (very simple) pod.

## Let's look at CNI

##


[my issue](https://github.com/Intel-Corp/multus-cni/issues/3)
[cni backend plugin specs](https://github.com/containernetworking/cni/tree/master/Documentation)
