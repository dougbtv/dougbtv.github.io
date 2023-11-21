---
author: dougbtv
comments: true
date: 2023-11-21 08:00:00-05:00
layout: post
slug: robocni-config
title: Using Robocniconfig to generate CNI configs with an LLM
category: nfvpe
---

Time to automate some secondary networks automagically by having configurations produced using a large language model (LLM)!

I saw ollama at Kubecon and I had to play with it :D It's really fun and I had good luck. I love the idea of having a containerized LLM and then hooking up my own toys to it! ...I've also played with doing such myself, maybe a little bit that I've mentioned on this blog or you've seen a `Dockerfile` or two from me. But! It's really nice to have an opinionated installation. 

Here's the concept of Robo CNI

* You (the user) give an instruction as a short blurb about what kind of CNI configuration you want, like "I'll take a macvlan CNI with an eth0 master, and IP addresses in the 192.0.2.0/24 range"
* RoboCNI runs that through a large language model, with some added context to help the LLM figure out how to produce a CNI configuration
* Then, we test that a bunch of times to see how effectively it works.

I'm doing something I heard data scientists tell me not do with it (paraphrased): "Don't go have this thing automatically configure stuff for you". Well... I won't be doing in production. I've had enough pager calls at midnight in my life without some robot making it worse, but, I will do it in a lab like... whoa!

So I wrote an application [Robo CNI Config](https://github.com/dougbtv/robocniconfig/)! It basically generates CNI configurations based on your prompts, so you'd use the core application like:

```
$ ./robocni --json "dude hook me up with a macvlan mastered to eth0 with whereabouts on a 10.10.0.0/16"
{
    "cniVersion": "0.3.1",
    "name": "macvlan-whereabouts",
    "type": "macvlan",
    "master": "eth0",
    "mode": "bridge",
    "ipam": {
        "type": "whereabouts",
        "range": "10.10.0.225/28"
    }
}
```

I basically accomplished this by writing some quick and dirty tooling to interface with it, and then a context for the LLM that has some CNI configuration examples and some of my own instructions on how to configure CNI. You can see [the context itself here in github](https://github.com/dougbtv/robocniconfig/blob/main/cmd/robocni/templates/base_query.txt).

It has stuff like you'd imagine something like ChatGPT is kind of "pre-prompted" with, like:

```
Under no circumstance should you reply with anything but a CNI configuration. I repeat, reply ONLY with a CNI configuration.
Put the JSON between 3 backticks like: ```{"json":"here"}```
Respond only with valid JSON. Respond with pretty JSON.
You do not provide any context or reasoning, only CNI configurations.
```

And stuff like that.

I also wrote a utility to automatically spin up pods given these configs and test connectivity between them. I then kicked it off for 5000 runs over a weekend. Naturally my kube cluster died from something else (VMs decided to choke), so I had to spin up a new cluster, and then start it again, but... It did make it through 5000 runs.

Amazingly: It was able to get a ping across pods that were automatically approximately 95% of the time. Way better than I expected, and I even see some mistakes that could be corrected, too.

It's kinda biased towards macvlan, bridge and ipvlan CNI plugins, but... You gotta start somewhere.

So, let's do it!

## Pre-reqs...

There's a lot, but I'm hopeful you can find enough pointers from other stuff in my blog, or... even a google search.

* A working Kubernetes install (and access to the kubectl command, of course)
* A machine with a GPU where you can run ollama
* [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) is installed, as well as [Whereabouts IPAM CNI](https://github.com/k8snetworkplumbingwg/whereabouts).

I used my own home lab. I've got a Fedora box with a Nvidia Geforce 3090, and it seems to be fairly decent for running LLMs and StableDiffusion training (which is the main thing I use it for, honestly!)

## Installing and configuring ollama

Honestly, it's really as easy as following [the quickstart](https://github.com/jmorganca/ollama#linux--wsl2).

Granted -- the main thing I did need to change was that it would accept queries from outside the network, so I did go and mess with the systemd units, you can find most of what you need in the [installation on linux](https://github.com/jmorganca/ollama/blob/main/docs/linux.md) section, and then also the port info [here in the FAQ](https://github.com/jmorganca/ollama/blob/main/docs/faq.md#how-can-i-expose-ollama-on-my-network).

I opted to use the LLaMA 2 model with 13B params. Use whatever you like (read: Use the biggest one you can!)

## Using RoboCNI Config

First clone the repo @ https://github.com/dougbtv/robocniconfig/

Then kick off `./hack/build-go.sh` to build the binaries.

Generally, I think this is going to work best on uniform environments (e.g. same interface names across all the hosts). I ran it on a master in my environment, which is a cluster with one 1 master and 2 workers.

You can test it out by running something like...

```
export OLLAMA_HOST=192.168.2.199
./robocni "give me a macvlan CNI configuration mastered to eth0 using whereabouts ipam ranged on 192.0.2.0/24"
```

That last part in the quotes is the "hint" which is basically the user prompt for the LLM that gets added to my larger context of what the LLM is supposed to do.

Then, you can run it in a loop.

But first! Make sure robocni is in your path.

```
./looprobocni --runs 5000
```

My first run of 5000 was.... Better than I expected!

```
Run number: 5000
Total Errors: 481 (9.62%)
Generation Errors: 254 (5.08%)
Failed Pod Creations: 226 (4.52%)
Ping Errors: 0 (0.00%)
Stats Array:
  Hint 1: Runs: 786, Successes: 772
  Hint 2: Runs: 819, Successes: 763
  Hint 3: Runs: 777, Successes: 768
  Hint 4: Runs: 703, Successes: 685
  Hint 5: Runs: 879, Successes: 758
  Hint 6: Runs: 782, Successes: 773
```

In this case, there's 5% generation errors, but... Those can be trapped. So discounting those... 95% of the runs were able to spin up pods, and... Amazingly -- whenever those pods came up, a ping between them worked.

Insert:`<tim-and-eric-mind-blown.jpg>`

I had more fun doing this than I care to admit :D
