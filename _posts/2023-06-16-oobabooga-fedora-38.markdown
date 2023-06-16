---
author: dougbtv
comments: true
date: 2023-05-06 08:06:00-05:00
layout: post
slug: oobabooga-fedora-38
title: Installing Stable Diffusion on Fedora 38 (with podman!)
category: nfvpe
---

Today we're going to run [Ooobabooga](https://github.com/oobabooga/text-generation-webui/) -- the text generation UI to run large language models (LLMs) on your local machine. We'll make it containerized so that you can keep everything sitting pretty right where it is, otherwise.

## Requirements

Looks like we'll need podman compose if you don't have it...

* Fedora 38
* A nVidia GPU
* Podman (typically included by default)
* podman-compose (optional)
* The nVidia drivers

If you want podman compose, pick up: 

```
pip3 install --user podman-compose
```

## Driver install

You're also going to need to install the nVidia driver, and the nVidia container tools

Before you install CUDA, do a `dnf update` (otherwise I wound up with mismatched deps), then install [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Fedora&target_version=37&target_type=rpm_local) (link is for F37 RPM, but it worked fine on F38)


And the container tools:

```
curl -s -L https://nvidia.github.io/libnvidia-container/centos8/libnvidia-container.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo dnf install nvidia-container-toolkit nvidia-docker2
```

(nvidia docker 2 might not be required.)

If you need more of a reference for GPUs on Red Hat flavored linuxes, [this article from the red hat blog is very good](https://www.redhat.com/en/blog/how-use-gpus-containers-bare-metal-rhel-8)

## Let's get started

In my experience, you've gotta use podman for GPU support in Fedora 38 (and probably a few versions earlier, is my guess). 

Go ahead and clone this [oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui)

From their [README](https://github.com/oobabooga/text-generation-webui#alternative-docker), you've gotta set this up to do the container build...

```
ln -s docker/{Dockerfile,docker-compose.yml,.dockerignore} .
cp docker/.env.example .env
# Edit .env and set TORCH_CUDA_ARCH_LIST based on your GPU model
docker compose up --build
```

*Importantly* -- you've got to set the `TORCH_CUDA_ARCH_LIST`. You can check that you've got the right one from [this grid on wikipedia](https://en.wikipedia.org/wiki/CUDA#GPUs_supported)
*DOUBLE CHECK* -- everything, but especially that you're using the right .env file. Because I really made that take longer than it should when I got that wrong.

```
TORCH_CUDA_ARCH_LIST=8.6+PTX
```

First, try building ti with podman -- it worked for me on the second attempt. Unsure what went wrong, but I built with...

```
podman build -t dougbtv/oobabooga .
```

*WARNING*: These are some BIG images. I think mine came out to ~16 gigs.

And then I loaded that image it into podman...

I need make a few mods before I can run it... Copy the .env file also to the docker folder (we could probably improve this with a symlink in an earlier step). And while we're here we'll need to copy the template prompts, presets, too.

```
cp .env docker/.env
cp prompts/* docker/prompts/
cp presets/* docker/presets/
```

Now you'll need at least a model, so to download one leveraging the container image...

```
podman-compose run --entrypoint "/bin/bash -c 'source venv/bin/activate; python download-model.py TheBloke/stable-vicuna-13B-GPTQ'" text-generation-webui
```

Naturally, change `TheBloke/stable-vicuna-13B-GPTQ` to whatever model you want.

You'll find the model in...

```
ls ./docker/models/
```

I also modify the docker/.env to change this line to...

```
CLI_ARGS=--model TheBloke_stable-vicuna-13B-GPTQ --chat --model_type=Llama --wbits 4 --groupsize 128 --listen
```

However, I run it by hand with:

```
podman run \
--env-file /home/doug/ai-ml/text-generation-webui/docker/.env \
-v /home/doug/ai-ml/text-generation-webui/characters:/app/characters \
-v /home/doug/ai-ml/text-generation-webui/extensions:/app/extensions \
-v /home/doug/ai-ml/text-generation-webui/loras:/app/loras \
-v /home/doug/ai-ml/text-generation-webui/models:/app/models \
-v /home/doug/ai-ml/text-generation-webui/presets:/app/presets \
-v /home/doug/ai-ml/text-generation-webui/prompts:/app/prompts \
-v /home/doug/ai-ml/text-generation-webui/softprompts:/app/softprompts \
-v /home/doug/ai-ml/text-generation-webui/docker/training:/app/training \
-p 7860:7860 \
-p 5000:5000 \
--gpus all \
-i \
--tty \
--shm-size=512m \
localhost/dougbtv/oobabooga:latest
```

(If you're smarter than me, you can get it running with podman-compose at this point)

**At this point, you should be done, grats!**

It should give you a web address, fire it up and get on generating!

### Mount your models somewhere

I wound up bind mounting some directories...

```
sudo mount --bind /home/doug/ai-ml/oobabooga_linux/text-generation-webui/models/ docker/models/
sudo mount --bind /home/doug/ai-ml/oobabooga_linux/text-generation-webui/presets/ docker/presets/
sudo mount --bind /home/doug/ai-ml/oobabooga_linux/text-generation-webui/characters/ docker/characters/
```

### Bonus note: I also wound up changing my dockerfile to install a torch+cu118, in case that helps you.

So I changed out two lines that looked like this diff:

```
-    pip3 install torch torchvision torchaudio && \
+    pip3 install torch==2.0.1+cu118 torchvision==0.15.2+cu118 torchaudio==2.0.2 -f https://download.pytorch.org/whl/cu118/torch_stable.html && \
```

I'm not sure how much it helped, but, I kept this change after I made it.

I'm hopeful to submit a patch for https://github.com/RedTopper/Text-Generation-Webui-Podman which isn't building for me right now hopefully integrating what I learned from this. And then have the whole thing in podman, later.

### Don't make my stupid mistakes

I ran into an issue where, I got:

```
RuntimeError: CUDA error: no kernel image is available for execution on the device
```

I tried messing with the TORCH_CUDA_ARCH_LIST in the .env file and change it to 8.6+PTX, 8.0, etc, the whole list, commented out, no luck.

I created an issue in the meanwhile: https://github.com/oobabooga/text-generation-webui/issues/2002 

### I also found this podman image repo!

https://github.com/RedTopper/Text-Generation-Webui-Podman

and I forked it.

It looks like it could possibly need updates.

I'll try to contribute my work back to this repo at some point.

