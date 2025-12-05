---
author: dougbtv
comments: true
date: 2025-12-05 10:00:00-04:00
layout: post
slug: comfyui-vllm-omni
title: Diffusion on vLLM-Omni?! Yup. Here’s How I Plugged It Into ComfyUI.
category: nfvpe
---

Ready to generate images using [vLLM-Omni](https://github.com/vllm-project/vllm-omni) for inference, but with [ComfyUI](https://github.com/comfyanonymous/ComfyUI) as the front-end? *Here we go!*

![example not quite tele skier](https://i.imgur.com/qeooWTu.png)
*Yup, that's a steam punk skier generated in ComfyUI with vLLM in the loop!*

I love myself a good diffusion model. Generative AI for imagery is what sparked my initial interest in the wave of gen AI, and I got really excited when I found out about [vllm-omni](https://github.com/vllm-project/vllm-omni) -- a framework that expands [vLLM](https://github.com/vllm-project/vllm) for *"omni-modality model inference"*, that is, other modes and in combinations of: text, images, video, as input and/or output. This also means for the first time that we can use diffusion models served from vLLM. Needless to say: I'm PUMPED. I'd been meaning to try the omni  model because it's been seeing a lot of hype in [/r/stablediffusion](https://www.reddit.com/r/StableDiffusion/) in the recent past.

I took the opportunity to build a PoC where I extended vllm-omni with an endpoint (modeled after the DALL-E OpenAI endpoint), so that I could then build a [ComfyUI custom node](https://docs.comfy.org/development/core-concepts/custom-nodes). We're going to have the vllm-omni portion deployed in docker, and you can deploy comfyUI wherever/however you'd like. I was originally inspired by the [offline inference examples for vLLM-omni](https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/examples/offline_inference/qwen_image/).

Why ComfyUI? Because it's THE serious tool for putting together workflows for diffusion models. It's got the production grade kind of things to allow you to compose and extend the UI. It isn't to say that there aren't other great front-ends for diffusion models, like, I love [automatic1111](https://github.com/AUTOMATIC1111/stable-diffusion-webui), but I want the ability to extend on this idea from Comfy, and it gives me a lot of freedom going forward to mix-and-match workflows in a composable fashion.

By the way -- the text in this blog article today is formed from "hand-typed, artisanal words," I wrote them myself because, while I'm using gen AI for *lots* of work, I really have a pride in my blog. Credit to my buddy [@csibbitt](https://github.com/csibbitt) for coining the phrase, "artisanal words". As for the code and examples, they're copy-pasta and iterated on, so at least the tech steps, partially by the [clankers](https://en.wikipedia.org/wiki/Clanker) (which sorta feels like a swear word, but, I'm not sure!).

I can't count the cups of coffee I had today (which means I can't count over 3), but, I'm just drinking long coffees today because it's like... part of the psychological pump up for putting together a PoC, gotta have a hot coffee at the ready.

## What do I need to get rolling?

Basic gist is that we're going to:
1. use Docker to run a build that I made of vLLM-omni, and
2. then we're going to install my custom ComfyUI node, comfyui-vllm-omni.

Alright -- the requirements are, well, fairly hefty. I'm using qwen3-image: https://huggingface.co/Qwen/Qwen-Image -- it might take more than what's in your prosumer system. 

* GPU(s) powerful enough to run qwen3-image and vLLM.
* An [installation of ComfyUI](https://docs.comfy.org/installation/manual_install) (manual installation linked, but  any will do)

I think that’s basically everything you need. I ran vLLM-Omni on a B200 system (overpowered for this model, honestly, but fun), and ComfyUI on my local workstation with a 50-series card (maybe later I’ll integrate it into a multi-step workflow with local inference).

As for limitations: It's 100% PoC mode. The API is "just a sketch", I figured following DALL-E's openai endpoint might make sense. Additionally, it kind of runs as a "stand alone example" in the vllm-omni code, it needs some considerations. For now the goal is "just a text2image API", but I'm also looking in image-to-image, even as I write (basically!). Also, I've only tested this with Qwen3-image, and  I'm not sure what else works, or doesn't...

## Quick tour of the assets...

* My [fork (and branch) of vLLM-omni](https://github.com/dougbtv/vllm-omni/tree/text-to-img-api)
	* Here's where I built out a new API endpoint that was based off of the examples for image generation using qwen3-image
* I have an image published: [quay.io/dosmith/vllm-omni:text2image-rev-b](https://quay.io/repository/dosmith/vllm-omni?tab=tags) 
	* (it's 12.5 gigs, heads up)
	* It's built from [the diffusion CI docker image](https://github.com/vllm-project/vllm-omni/blob/main/docker/Dockerfile.ci)
	* Of interest: This uses a vllm-openai base image!
* My custom ComfyUI node, [ComfyUI-vLLM-Omni Text-to-Image Node](https://github.com/dougbtv/comfyui-vllm-omni)
	* The basic gist is that it's... Fairly bare bones for now. It gets you a way to query the custom endpoint I made in vllm-omni.

## Firing up vLLM-omni

First, download your qwen-image, and make note of your Hugging Face cache directory...

```
hf download Qwen/Qwen-Image
```

Then, you can run from my image...

```
  docker run -it --rm --gpus all \
    -p 8000:8000 \
    -e NVIDIA_VISIBLE_DEVICES=0 \
    -e CUDA_VISIBLE_DEVICES=0 \
	-v /path/to/your/hub_cache/:/root/.cache/huggingface \
    quay.io/dosmith/vllm-omni:text2image-rev-b \
    python -m vllm_omni.entrypoints.openai.serving_image \
      --model Qwen/Qwen-Image \
      --port 8000
```

Take note: I'm using `CUDA_VISIBLE_DEVICES=0` because it’s a shared system and I don’t want to accidentally grab another user’s GPU. You can tailor or remove these settings as needed.. Additionally, I was on the (gotta admit, weird) Nvidia ubuntu distro on this b200. So I was using docker and not podman. You can convert it easily.

From there, now you can generate an image using a REST endpoint, like this...

```
curl -X POST http://localhost:8000/v1/images/generations \
    -H "Content-Type: application/json" \
    -d '{
      "prompt": "some total vermont ski dude",
      "size": "1024x1024",
      "seed": 42
    }' | jq -r '.data[0].b64_json' | base64 -d > skidude.png
```

There's not a lot in terms of... progress or stats yet. But, we'll get there.

That's all you need on the vLLM-omni side.

## Installing the custom node

Quite easy, what you'll do is navigate to the `custom_nodes` folder in your ComfyUI deployment (in my case, from a git clone on my workstation), and then clone my [ComfyUI-vLLM-Omni Text-to-Image Node](https://github.com/dougbtv/comfyui-vllm-omni)

```
cd ./ComfyUI/custom_nodes
git clone git@github.com:dougbtv/comfyui-vllm-omni.git
```

From here, you might need to restart ComfyUI to see it, but then you can add the node.

In my case I browse to the `Node Library` on the left nav (looks like a chain link) and then search for `vllm` and you'll find  the node there.

Update the `server_url` with the correct address for where you vLLM-omni instance is running, and set other parameters as you wish. 

Also of note,  `1328x1328`, `1664x928`, `928x1664`, `1472x1140`, `1140x1472`, `1584x1056`, and `1056x1584` are the sizes expected to give the best result on these dimensions for qwen-image.

You can also start with [this example JSON workflow I posted on github gists](https://gist.github.com/dougbtv/460bf12fe79cc24aca282022f5475be4), too.

And soon! You should be here.

![I leave you with this](https://i.imgur.com/PR78eUj.png)

What's next?! ...I might see if I can get an image-to-image workflow, and I'm looking forward to engaging the community to find out what's next for the project.
