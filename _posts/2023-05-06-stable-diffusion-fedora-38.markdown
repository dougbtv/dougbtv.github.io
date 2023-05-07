---
author: dougbtv
comments: true
date: 2023-05-06 08:02:00-05:00
layout: post
slug: stable-diffusion-fedora-38
title: Installing Stable Diffusion on Fedora 38
category: nfvpe
---

I'm putting together a lab machine for GPU workloads. And the first thing I wanted to do was get Stable Diffusion running, and I'm also hopeful to start using it for training LoRA's, embeddings, maybe even a fine tuning checkpoint (we'll see).

Fedora is my default home server setup, and I didn't find a direct guide on how to do it, although it's not terribly different from other distros

...Oddly enough I actually fired this up with Fedora Workstation.

## Installing Automatic Stable Diffusion WebUI on Fedora 38

I'm going to be using Vladmanic's fork of [Automatic1111 sd webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui): [https://github.com/vladmandic/automatic](https://github.com/vladmandic/automatic)

Clone it.

Fedora 38 ships with Python 3.11, but some dependency for stable diffusion requires python 3.11, which will require a few extra steps.

Install python 3.10

```
dnf install python3.10
```

Also, before you install CUDA, do a `dnf update` (otherwise I wound up with mismatched deps for NetworkManager and couldn't boot off a new kernel, and I had to wheel up a [crash cart](https://en.wikipedia.org/wiki/Crash_cart#In_computing), just kidding I don't have a crash cart or a KVM for my Linux lab so it's much more annoying where I move my server to my workstation area, luckily I just have a desktop server lab)

Install [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Fedora&target_version=37&target_type=rpm_local) (link is for F37 RPM, but it worked fine on F38)

And -- follow the instructions there. You might need to reboot now.

Make a handler script to export the correct python version... I named mine `user-webui.sh`


```
#!/bin/bash
export python_cmd=python3.10
screen -S ./webui.sh --listen
```

NOTE: I fire it up in `screen`. If you don't have Stockholm Syndrome for screen you can decide to not be a luddite and modify it to use `tmux`. And if you need a [cheat sheet for screen](https://gist.github.com/jctosta/af918e1618682638aa82), there you go. I also use the `--listen` flag because I'm going to connect to this from other machines on my network. 

Then run `./user-webui.sh` the script once to get the venv, it may likely fail at this point.

Then enter the venv.

```
 . venv/bin/activate
```

Then ensurepip...


```
python3.10 -m ensurepip
```

And now you can fire up the script!

```
./webui.sh
```
