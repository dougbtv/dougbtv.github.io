---
author: dougbtv
comments: true
date: 2023-04-14 08:01:00-05:00
layout: post
slug: oobabooga-api
title: Running LLM's locally and interacting with an API (Using Ooobabooga Web UI)
category: nfvpe
---

Have you played with ChatGPT yet? Ummm, yeah, who hasn't!? I have pirate-styled rap battles to make! So let's get right to the point so we can get back to generating rap-battles as soon as possible!

Today we're going to run a LLM (Large Language Model) locally on one of our own machines, and we're going to set it up so that we can interface with it via API, and we'll even write a small program to test it out.

I have some ideas where I want to take some software I'm building and hook it up to one of these, later I'd like to train it on custom data and then query it. Maybe even have some like real-ish-time data fed to it and then be able to query it in near-real-time too. I also have some ideas for populating it with logs and then being like "yo, tell me what's up with this machine?"

But yeah, instead of relying on a GPT service, I want to run the LLM myself, using open source tools.

## Pre-requisites

Ok, so I'm not going to go deep into the details of the installation, so I'm just going to give some pointers. It's not necessarily rocket science

First up, we're going to install a webUI, OobaBooga: [https://github.com/oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui)

This is one of the few times I'm going to say the word "windows" on this blog, but, I actually installed mine on windows, because it's a windows box that's an art and music workstation where I've got my decent GPU (for gaming and also for stable diffusion and then associated windoze-y art tools). I follow [this youtube video](https://www.youtube.com/watch?v=ByV5w1ES38A&t=191s) by [@TroubleChute](https://www.youtube.com/@TroubleChute). I even used his opinionated script to automatically install Vicuna!

But you can also just install it with [the instructions on the README](https://github.com/oobabooga/text-generation-webui#installation), which appears to be really straight forward.

The Model we're going to use, Vicuna, you can find it @ [https://vicuna.lmsys.org/](https://vicuna.lmsys.org/) -- the thing that's interesting about Vicuna is that it's trained on crowd-sourced GPT output, and claims be 90% as good as GPT, which seems like a lofty statement. But so far it does seem like it's pretty decent, even if it does parrot a lot of the kind of "walled garden" stuff that ChatGPT says ("As an AI language model, I can't tell you what you asked me for" kind of stuff.)

*Quick tip*: After you install it and you're playing with it on the WebUI, assuming you've installed Vicuna... Go to the bottom of the chat and find the radio button for "mode" and switch it to "instruct" and then in the "instruction template" drop down, select "Vicuna" -- this will parse the output from the LLM so it makes more sense in context with your queries/conversation/prompts to it.

## Configure the start up script to launch the API.

Well, first, in your text-generation-webui git clone -- go and check out the `./extensions/api` folder and make sure there's a script there, there should be by default.

Next, we're going to enable that by adding a flag to our startup script.

In your startup script, add a `--extensions api` field, and also a `--listen` flag (if you want to listen on all addresses and not just your loopback so you can access the api from your LAN/WAN). 

So now, for example, mine looks like:

```
python server.py --extensions api  --listen --auto-devices --chat --model anon8231489123_vicuna-13b-GPTQ-4bit-128g --wbits 4 --groupsize 128
```

Now your api should be ready to query.

## Calling the API

Then I whipped together a node.js script (with ChatGPT's help!) to query the API.

I've got [the script here in this github gist](https://gist.github.com/dougbtv/249a31e0d9d671715394a99139a5d3bd).

Lucky you, the script asks for suggestions for cheese. If you ask me, I'd tell you have to have a [Bayley Hazen blue](https://www.jasperhillfarm.com/bayley), or maybe the [Cabot Clothbound](https://www.jasperhillfarm.com/clothbound) both from Jasper Hill Farms (they've go the raddest cheese cave, super good cheese)

And you can just run that with

```
node oobabooga_api_query.js
```

And away you go. Or if you don't like Node, just have ChatGPT re-write it for you in assembly or something!

(The script still needs a little help parsing, but it's close! I think it's kind of missing the kind of "mode" + "template" from the UI, so that's next to figure out)


## B-b-b-bonus hip hop battle

Just for you, I asked it to:

```
write a rap battle about api clients written in node.js vs. assembly
```

And it gave me:

```
Verse 1 (Node.js):

I'm Node, I'm fast and sleek,
When it comes to APIs, I can't be beat!
With JavaScript at my core,
I make building apps a breeze, no need to feel bore-ot.

Chorus:

API clients, we rule the land,
Node.js or Assembly, which one takes the stand?
We may not be as old school,
But when it comes to speed and coolness, we'll never lack cool.

Verse 2 (Assembly):

I may be older than you think,
But don't underestimate me, I still have some tricks.
With raw power and precision,
My code runs smoothly without any friction.

Chorus:

API clients, we rule the land,
Node.js or Assembly, which one takes the stand?
We may not be as old school,
But when it comes to speed and coolness, we'll never lack cool.

Outro:

In the end, it doesn't matter who wins this battle,
Because both Node.js and Assembly are great for different purposes, so let's celebrate!
```

Thanks for your "everyone gets an award!" style indifference LLM! 

...But not too shabby for something running on my local!