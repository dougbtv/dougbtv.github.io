---
author: dougbtv
comments: true
date: 2025-01-13 13:00:01-04:00
layout: post
slug: vllm-omni-ltx2
title: An Experimental LTX-2 Integration for vLLM-Omni (Expect Weird Videos)
---

Well, doggies! When the [LTX-2 video model](https://huggingface.co/Lightricks/LTX-2) landed last week, I went ahead and brewed up an experimental implementation of it in [vllm-omni](https://github.com/vllm-project/vllm-omni), the omni-modality serving layer for [vLLM](https://github.com/vllm-project/vllm).

LTX-2 was [released on January 6th](https://ltx.io/model/model-blog/ltx-2-is-now-open-source), and I wanted to see how far I could get wiring it up end-to-end. From [the HuggingFace repo](https://huggingface.co/Lightricks/LTX-2) (emphasis mine)...

> LTX-2 is a DiT-based audio-video foundation model designed to generate synchronized video and audio within a single model. It brings together the core building blocks of modern video generation, with **open weights** and a focus on practical, local execution.

I was going to try it out in ComfyUI, but I decided "hey maybe I can brew up an implementation based on the [Lightricks/LTX-2 GitHub repo](https://github.com/Lightricks/LTX-2)", so I got to it.

Today for you -- I've got a Docker image published with my work, up on [quay.io/dosmith/vllm-omni:ltx2-experimental](https://quay.io/repository/dosmith/vllm-omni?tab=tags) and instructions to fire it up, and then followed by some notes on the implementation.

Since I've been experimenting with vllm-omni recently, I wanted to see if I could brew up an implementation -- I've at least got it producing video (no audio! I wanted to PoC it before I tackled it):

<figure>
  <div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
    <iframe
      src="https://www.youtube.com/embed/SbAoA1m35eM"
      title="YouTube Short"
      frameborder="0"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen
      style="position:absolute;top:0;left:0;width:100%;height:100%;">
    </iframe>
  </div>
  <figcaption>
    Demo Short:
    <a href="https://youtube.com/shorts/SbAoA1m35eM">Watch on YouTube</a>
  </figcaption>
</figure>
Actually that was the first thing I got to come out of it, basically (well, at least the first working thing that I intentionally prompted for!)

This one came out a little better, but it's not quite as fancy schmancy as the videos on [the ComfyUI support announcement](https://blog.comfy.org/p/ltx-2-open-source-audio-video-ai) .

<figure>
  <div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
    <iframe
      src="https://www.youtube.com/embed/kSpiqg7hsjU"
      title="YouTube Short"
      frameborder="0"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen
      style="position:absolute;top:0;left:0;width:100%;height:100%;">
    </iframe>
  </div>
  <figcaption>
    Demo Short: 
    <a href="https://youtube.com/shorts/kSpiqg7hsjU">Watch on YouTube</a>
  </figcaption>
</figure>
Also, I tried to do the ["infamous Will Smith eating spaghetti" (wikipedia page)](https://en.wikipedia.org/wiki/Will_Smith_Eating_Spaghetti_test) (holy smokes this has a wiki article!? I thought it was a /r/StableDiffusion meme!). Turns out... I suspect the model must've been scrubbed for copyright infringement type problems, and... It didn't look like Will Smith at all (so I omitted posting it!).

These 1024x1024 10-second videos took me about 3 minutes end-to-end to run vLLM-omni and produce, on a single A100 GPU.

If you stay until the end, I'll even show you one I generated that was inspired by [the Tim & Eric Cinco I-Jammer](https://www.youtube.com/watch?v=NTNXQH8SEMo).

## Alright... Well, how do I run it!?

A few quick notes about how and where I ran it, and a few of the limitations before I get into it...

I ran it:
* On an NVIDIA A100 (80G of VRAM)
* Using Podman
* On RHEL 9.6

But other than that, it should be straightforward...

```bash
# Set your HuggingFace token
export HF_TOKEN=hf_your_token_here

podman run --rm \
  --device nvidia.com/gpu=0 \
  --security-opt=label=disable \
  --userns=keep-id \
  -e NVIDIA_VISIBLE_DEVICES=0 \
  -e CUDA_VISIBLE_DEVICES=0 \
  -e HF_TOKEN="${HF_TOKEN}" \
  -e HUGGINGFACE_HUB_CACHE=/hf/hub \
  -v ~/.cache/huggingface:/hf/hub \
  -v $(pwd)/outputs:/output \
  quay.io/dosmith/vllm-omni:ltx2-experimental \
  python examples/offline_inference/text_to_video_ltx2.py \
    --model Lightricks/LTX-2 \
    --prompt "A corny used-car-lot commercial set in a dusty western town" \
    --height 1024 --width 1024 --num_frames 121 \
    --fps 24.0 \
    --output /output/commercial.mp4
```

Note: You could probably use plain old `docker` too, but might require munging that command a little bit (GPU flags, the SELinux bits, etc.)
## Where does it live?

I've got the container image posted on quay.io for you:

* [quay.io/dosmith/vllm-omni:ltx2-experimental](https://quay.io/repository/dosmith/vllm-omni?tab=tags)

And I've got the code in my branch of vllm-omni, on GitHub:

* [dougbtv/vllm-omni](https://github.com/dougbtv/vllm-omni/tree/ltx2) in the `ltx2` branch.

I've got a `docker/Dockerfile.ltx2` there with how I built it as well.
## How I wound up getting there

This work is primarily meant to be experimental. I wanted to see if I could "make it work" so that I could try out both LTX-2, and adding a new model implementation in vLLM-Omni. I'm not sure where it stands in terms of permanence.

I used both the LTX-2 GitHub repo, and the ComfyUI implementation as references. Then used a combination of ChatGPT (with GPT-5.2) to orchestrate prompts for both Claude as well as some back and forth with Codex to check the work. I used these tools to then leverage the LTX-2 code primarily, with the LTX‑2 reference stack via the `ltx_core` + `ltx_pipelines` libraries, which provide the model components (transformer, VAE, schedulers, helpers) used by the vLLM‑Omni LTX‑2 pipeline in [my branch](https://github.com/dougbtv/vllm-omni/tree/ltx2).

In summary from Codex...

- **Introduced an LTX‑2 diffusion pipeline** with model loading, distilled 8‑step denoising, video‑only (no audio), and post‑processing.
- **Added LTX‑2 configuration helpers** to enforce dimension constraints (32‑pixel multiples + 8K+1 frames) and provide defaults for fps, inference steps, and frame count.
- **Registered LTX‑2 in the diffusion registry** and wired up a dedicated post‑process function for decoding video outputs.
- **Shipped a detailed offline example** that shows how to run LTX‑2, extract pipeline outputs, and export an MP4 using PyAV.

## Debugging: the VAE problem

The initial implementation seemed feasible, but... I ran into some issue with what I recognized was probably the VAE decoding. How'd I know that? Well, if you were fiddling around with StableDiffusion 1.X when it was first coming out, you'd see this often if you misconfigured something. Also -- you'd see people post similar problems on /r/stablediffusion in those days.

For an example, you can see some of [the busted video here on imgur](https://imgur.com/NZe2oIX)

![the busted frame](https://i.imgur.com/IlDjpoJ.png)

Luckily, my gut was right, turns out I had the VAE decoding in the wrong spot, order-of-operations-wise.

This brought me to the point where I could make the kind of unhinged things that I like to make such as the...
## Bonus: Cowboy O-Hungee

You stayed the entire blog article... and your reward is... This!? Sorry.

<figure>
  <div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
    <iframe
      src="https://www.youtube.com/embed/CLe_OnC0NoQ"
      title="YouTube Short"
      frameborder="0"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen
      style="position:absolute;top:0;left:0;width:100%;height:100%;">
    </iframe>
  </div>
  <figcaption>
    Demo Short: 
    <a href="https://youtube.com/shorts/CLe_OnC0NoQ">Watch on YouTube</a>
  </figcaption>
</figure>

