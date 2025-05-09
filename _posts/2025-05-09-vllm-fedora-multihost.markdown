---
author: dougbtv
comments: true
date: 2025-05-09 9:05:00-05:00
layout: post
slug: vllm-fedora-multihost
title: vLLM at Home -- Distributed Inference with Fedora and a 50-Series GPU
category: nfvpe
---

I've been pretty excited to say hello to [vllm](https://github.com/vllm-project/vllm), which is a library for LLM serving and inference. That along with some recent upgrades in my lab setup, I figured it was time to do it in the way that best fit my own brand, with these parameters:

* Running on Fedora
* Containerized (using Podman)
* Using my new 50-series card in my new desktop
* And... Distributed inference across my desktop and my GPU lab server.

It was *fun* to do, and satisfying! I learned a lot and I'm hopeful to be able to contribute something back to the community based on what I learned. Quick note that... This is totally what I'd call a "toy setup". If I learned anything, it's that vLLM has a design that's meant for industrial-type applications, like, in the DCs. The kind of place where I used to drink burnt coffee at 2am in my telco switch tech days. But for this one -- we can drink legit espresso at home and kick it laid back in some [Thai fisherman pants](https://en.wikipedia.org/wiki/Fisherman_pants). If you're following along, I recommend a [flat white](https://en.wikipedia.org/wiki/Flat_white) for this one, I've been making mine using cream-top milk from [Sweet Rowen Farmstead](https://www.sweetrowen.com/products) here in Vermont. It's got a high milk fat content, so it basically doesn't foam at all.

I'm inspired in part by [this YT video about running an LLM across multiple computers](https://www.youtube.com/watch?v=ITbB9nPCX04). With the work I've been doing in exploring [Dynamic Resource Allocation (DRA) in Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/), this really speaks to me. And! I've got a good lab setup for it, too. I like this demo because it used prosumer hardware and it was well put together. However! It doesn't use a containerized setup. I'm really interested in the distributed aspect, because I'm also curious how this might impact K8s networking in the future, too. Last but not least I recently made an upgrade in my lab, so I've got 30, 40, and 50-series Nvidia cards. I had a hard time deciding if I should buy a 50-series, but I found a good deal on a 5060ti and was pretty excited it had a good chunk of VRAM, I kept wondering if it'd be worth it. Well, I have to say, with what I was able to learn using it, already worth it.

## The tl;dr

**Goal**: Run vLLM in my home lab with distributed inference across two machines using a 50-series NVIDIA GPU (`sm_120` arch).

**The official image doesn't work with 50-series GPU yet**, due to unsupported CUDA arch, I built a custom Docker image using CUDA 12.9 and compiled with `sm_120` support.

I **used Podman on Fedora**, with `nvidia-container-toolkit`, and CDI setup for GPU access.

**Distributed inference handled via Ray**; using `--net=host` for simplicity in a lab environment.

**Built image**: [dougbtv/vllm](https://hub.docker.com/repository/docker/dougbtv/vllm/general) (~26GB)

**Bonus**: Played with structured outputs -- super awesome.

**Next steps**: Might see what I can contribute back to vLLM, especially on the tooling and CI front.

...That being said, grab your flat white and let's hit the terminal...


## Getting my bearings

My initial instinct was to build an image from the `./docker/Dockerfile.nightly_torch` in a clone of [vllm](https://github.com/vllm-project/vllm), because I liked the way it read the most of the Dockerfiles, especially because I was worried with a 50-series, I might have to use something poppin' fresh. 

But, in the meanwhile I tried to use their official Docker image on my desktop workstation.

### However, I had to setup CDI, first.

I ran into this on a `docker run`...

```
Error: setting up CDI devices: unresolvable CDI devices nvidia.com/gpu=all
```

So I am reminded of [the Nvidia container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#with-dnf-rhel-centos-fedora-amazon-linux), and [the Nvidia container tool kit docs on CDI]().

```
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
# This was unnecessary, and didn't work with dnf5
# sudo dnf config-manager --enable nvidia-container-toolkit-experimental
sudo dnf install -y nvidia-container-toolkit
# Don't forget to generate the CDI manifest!
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
# Unsure if I needed to, but, out of instinct I did...
sudo systemctl restart podman
```

If you're not familiar with CDI, it's the "container device interface", and [the CNCF container-device-interface repo](https://github.com/cncf-tags/container-device-interface) is probably the place to go to learn more, but it allows us to allocate the GPU for our container to use.

### Using the official vLLM Docker image

I picked up [the docs for using vllm's official docker image](https://docs.vllm.ai/en/stable/deployment/docker.html), which I think -- in most cases, this will probably work with more hardware, but as you'll see.

And then I pulled with:

```bash
podman pull vllm/vllm-openai:latest
```

And with that, I also looked at [the vLLM distributed serving docs](https://docs.vllm.ai/en/latest/serving/distributed_serving.html).

The docs recommend this run command, for podman...

```
podman run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model mistralai/Mistral-7B-v0.1
```

Also, their example uses [Mistral 7B v0.1](https://huggingface.co/mistralai/Mistral-7B-v0.1), which I needed access for, so, make sure your hugging face token is there, apparently you have to agree to share some info with the authors to download it. So I did that on hugging face, and got it to run, but was met with...

```
NVIDIA GeForce RTX 5060 Ti with CUDA capability sm_120 is not compatible with the current PyTorch installation.
The current PyTorch install supports CUDA capabilities sm_50 sm_60 sm_70 sm_75 sm_80 sm_86 sm_90.
```

## Getting an `sm_120` compatible build

By `sm_120`, I'm referring to the compute capability of NVIDIA Blackwell architecture (that is, the RTX 50-series GPUs).

Now it's time to roll up our sleeves and make a full build. In the end, I used this modified [Dockerfile](https://gist.github.com/dougbtv/d94ce156654462c02b5dfaf885a38479), more on my process as you read through. I did bump CUDA to 12.9 from 12.8, and I also had to make sure I specified the new `sm_120` arch, too.

Mostly, I could just use:

```
podman build \
  -t dougbtv/vllm \
  -f docker/Dockerfile.nightly_torch .
```

I got a hint from my bud [Russell](https://github.com/russellb) about `VLLM_USE_PRECOMPILED=1`, which uses [vLLMs prebuilt wheels](https://docs.vllm.ai/en/latest/getting_started/installation/gpu.html#pre-built-wheels). I get a build after about... 30-45ish minutes? Nice way to toast up my CPU and let my AIO cooler purr. During the build I see a lot of build instructions with `sm_*` stuff but not `sm_120` which had me concerned, and, I didn't get a copy/paste because it exceeded my terminal buffer, oh well! I wound up with a number of attempts, and was getting stuff when I tried to `podman run` it, like:

```
# vllm serve --model mistralai/Mistral-7B-v0.1
[...snip...]
ModuleNotFoundError: No module named 'vllm._C'
```

I found a (kinda old) [github issue with a reference to this error](https://github.com/vllm-project/vllm/issues/1814), but, I had a feeling it had something to with how I built it.

Typically, I'd be running it like this, and also referencing [the docs for the engine arguments](https://docs.vllm.ai/en/latest/serving/engine_args.html#engine-args):

```
podman run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
  --ipc=host \
  localhost/dougbtv/vllm:latest \
  mistralai/Mistral-7B-v0.1
```

I figure out that I missed a major thing -- the arch list for `12.0`, aka `sm_120`

So, I need stuff like:

```Dockerfile
ARG torch_cuda_arch_list='8.0;8.6;8.9;9.0;12.0'
# and...
ARG vllm_fa_cmake_gpu_arches='80-real;90-real;120-real'
```

During the build -- I had a couple unlucky crashes (which I didn't go deeply into to diagnose, new F42 and all that), but! I got a build. Of course I forgot to time it, it happened overnight. I also need to figure out the caching cause I don't think I'm caching and the Dockerfile sure looks like it has a lot of hooks for caching, and it must be SUPER important with these huge builds.

I posted [my originally working Dockerfile as a GitHub gist](https://gist.github.com/dougbtv/d94ce156654462c02b5dfaf885a38479) and a diff of it (used against commit `6115b115826040ad1f49b69a8b4fdd59f0df5113`, but! Be warned, I think at some point in the Dockerfile build process it pulls a fresh head from a git remote).

Some hiccough prevented me from logging into quay because of some weirdness, so I pushed...

```bash
docker.io/dougbtv/vllm
```

Which you can find [on Dockerhub as dougbtv/vllm](https://hub.docker.com/repository/docker/dougbtv/vllm/general). It's about 26 gigs, so, [buyer beware](https://en.wikipedia.org/wiki/Caveat_emptor).

## Getting vLLM running on a single host and giving it a test.

And I run it like this:

```bash
podman run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --env HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
  --env PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --ipc=host \
  localhost/dougbtv/vllm:latest \
  mistralai/Mistral-7B-v0.1 \
  --gpu-memory-utilization 0.5
```

I'm getting a lot further, but I see:

```
torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 112.00 MiB. GPU 0 has a total capacity of 15.47 GiB of which 68.00 MiB is free. Including non-PyTorch memory, this process has 12.51 GiB memory in use.
```

I tried to look around quite a bit... But I realized something VERY important. Here's the thing:

vLLM isn't ollama. vLLM is meant for high-throughput inference serving in data centers, not the scrappy-style local inference that I usually do. So, something like ollama uses some quantized GGUF models and CPU offloading to work in constrained environments, but, this is designed differently.

I previously assumed I could run a model that sized on this 5060ti with 16 gigs of VRAM, but, not this way. So I decided to run a tiny model instead, `facebook/opt-125m`

```
podman run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --security-opt=label=disable \
  --device nvidia.com/gpu=0 \
  --env HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
  --env PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --ipc=host \
  localhost/dougbtv/vllm:latest \
  facebook/opt-125m \
  --gpu-memory-utilization 0.8
```

And it "works"!

```bash
 curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "prompt": "What is the capital of Vermont?",
    "max_tokens": 20
  }'
{"id":"cmpl-b062e3b04f48487590b36e21cfba0839","object":"text_completion","created":1746708569,"model":"facebook/opt-125m","choices":[{"index":0,"text":"  Edit: Or, I heard Vermont is a home state\nI don't think you put the","logprobs":null,"finish_reason":"length","stop_reason":null,"prompt_logprobs":null}],"usage":{"prompt_tokens":8,"total_tokens":28,"completion_tokens":20,"prompt_tokens_details":null}}
```

Maybe just "Montpelier" would be an appropriate answer.

I do think I learned something here though...

But, like [The Bad News Bears](https://en.wikipedia.org/wiki/The_Bad_News_Bears), sometimes I just gotta try it, put the heart into it. And that part worked! I got the 50-series working with vLLM which was half the goal.

I tried loading a quantized model, took me a few tries to get the params right...

```bash
podman run \
  --gpus all \ 
  -v /home/doug/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --env HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
  --env PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --ipc=host localhost/dougbtv/vllm:latest \
  TheBloke/Mistral-7B-Instruct-v0.1-AWQ \
  --quantization awq \
  --dtype float16 \
  --gpu-memory-utilization 0.95  
```

Which also appears to work...

```bash
curl http://localhost:8000/v1/completions   -H "Content-Type: application/json"   -d '{
    "model": "TheBloke/Mistral-7B-Instruct-v0.1-AWQ",
    "prompt": "What is the capital of Vermont?",
    "max_tokens": 20
  }'
{"id":"cmpl-4c7d34a0a8a547d4b4fffbaa2b64fdfa","object":"text_completion","created":1746721156,"model":"TheBloke/Mistral-7B-Instruct-v0.1-AWQ","choices":[{"index":0,"text":"\nMontpelier","logprobs":null,"finish_reason":"stop","stop_reason":null,"prompt_logprobs":null}],"usage":{"prompt_tokens":9,"total_tokens":14,"completion_tokens":5,"prompt_tokens_details":null}}
```

Great. Nailed it this time. 

### And give it a run on the 3090 machine...

Now on the 3090 machine, which I call `stimsonmt`. It's named for [a nearby mountain to my house](https://www.peakbagger.com/peak.aspx?pid=25364), it sits next to another box `bonemt` which is on the other side of the valley, so, that way I know which is which positionally, as they're pets! Fun fact, [Whereabouts IPAM CNI](https://github.com/k8snetworkplumbingwg/whereabouts)'s topographic logo is based on the topo for Stimson Mt!

Going to try with the example mistral model from the docs, again...

```bash
podman run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --env HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
  --env PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --security-opt=label=disable \
  --device=nvidia.com/gpu=all \
  --ipc=host \
  docker.io/dougbtv/vllm \
  mistralai/Mistral-7B-v0.1 \
  --gpu-memory-utilization 0.95
```

I had to add `--device=nvidia.com/gpu=all` certainly because of how I had setup podman on this machine (I went and scooped the podman run that I'm using for kohya_ss! so I had a good reference). I'll try to normalize them later.

Appears to run... let's try it...

```bash
curl http://localhost:8000/v1/completions   -H "Content-Type: application/json"   -d '{
    "model": "mistralai/Mistral-7B-v0.1",
    "prompt": "What is the capital of Vermont?",
    "max_tokens": 20
  }'
{"id":"cmpl-fa383b641f79470495d3e9a0fa537c25","object":"text_completion","created":1746725937,"model":"mistralai/Mistral-7B-v0.1","choices":[{"index":0,"text":"\n\nThe capital of Vermont is Montpelier, which is also the state's smallest","logprobs":null,"finish_reason":"length","stop_reason":null,"prompt_logprobs":null}],"usage":{"prompt_tokens":9,"total_tokens":29,"completion_tokens":20,"prompt_tokens_details":null}}
```

Took 4 tries before it got the right answer, but it did work! That used basically the whole GPU VRAM, too.

```
doug@stimsonmt:~$ nvidia-smi --query-gpu=memory.total,memory.free --format=csv
memory.total [MiB], memory.free [MiB]
24576 MiB, 824 MiB
```

## B-b-b-bonus round: Structured output!

While I was at this point in the process, I wound up joining [the vLLM office hours](https://neuralmagic.com/community-office-hours/) and listened to [Russell](https://github.com/russellb) talk about [structured output](https://docs.vllm.ai/en/v0.8.2/features/structured_outputs.html), so I also tried that out.

Structured output is a process to kind of mold the LLM to give you output in a specific structure, like, say you want a return in all JSON, or a specific set of values, heck [you can even use a regex](https://xkcd.com/208/)!

I brewed up [this python script](https://gist.github.com/dougbtv/fb4415371e979388cf93193f40f0d870) (as the curl was getting out of hand), and made a script to determine if the sentiment is "rad" or "bogus" depending on what you feed in, so I also tried that...

```
$ python test.py "the powder is wicked in the trees dude"
rad
$ python test.py "parcheesy is the best game"
bogus
```

## Getting it running across nodes.

Now that I had a baseline, I went to see if I can run it on two nodes...

After refreshing myself on a few moments in the YT video -- I realize that there's one thing I'm probably going to have to account for for now, you have to **specify an interface** as some parameters.

This makes me feel a few ways:

* It's so network infra kinda stuff: Specifying the interface.
* This has been a decade long battle in container land.
* ...I know I've got this one on lock, one way or another *\*puts shades on\**

We're going old school, where it all begain for me with containers and networking...

```
--net=host
```

For this scenario, it's probably the most efficient, and security isn't my biggest concern. As per usual, I have a long story about how I used to use this, and now I don't, and I feel like I was instrumental in making it so that people don't have to do that anymore, but for today? It's just the thing. Maybe we can return to this to handle it another way, but it's also kind of its own project.

You'll notice here that we're using [ray-project/ray](https://github.com/ray-project/ray), `ray`:

> Ray is a unified framework for scaling AI and Python applications

Sounds just like what we want to do, awesome.

Ok that's a good start... `ray` is already built in our image, phew!

```
root@478a2ec83613:/vllm-workspace# ray --version
2025-05-09 05:04:41,498 - INFO - Note: NumExpr detected 24 cores but "NUMEXPR_MAX_THREADS" not set, so enforcing safe limit of 16.
2025-05-09 05:04:41,498 - INFO - NumExpr defaulting to 16 threads.
ray, version 2.46.0
```

Unfortunately, `iproute2` isn't installed, so we have to install it with `apt install iproute2`, but I wasn't quite expecting it -- but I see the host interface inside the pod. 

```
root@e60d5253152e:/vllm-workspace# ip a | grep -P "^\d|192"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65520 qdisc fq_codel state UNKNOWN group default qlen 1000
    inet 192.168.50.198/24 brd 192.168.50.255 scope global noprefixroute eno1
```

And I can inspect it with:

```
$ podman inspect --format '{{.HostConfig.NetworkMode}}' condescending_mendel
pasta
```

Apparently! I'm not familiar with the ins-and-outs of `pasta` networking from podman. [These docs](https://github.com/eriksjolund/podman-networking-docs) provide a pretty good overview, it's named that way for "Point-to-point Adaptive Shared Transport Architecture". I'm so used to CNI. Sooooo! I'm not sure of the risks, so, we're sticking with host networking.

Here's a run with host networking that I'm using for checking the situation out...

```bash
podman run -it --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  --env HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
  --env PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  --net=host \
  --ipc=host \
  --entrypoint=/bin/bash localhost/dougbtv/vllm:latest
```

I'm going to have my `stimsonmt` host act as the "head", the main node, the only reason I chose it is because it has more VRAM, having a 3090. My other host (named: `yoda`, long story on that one) will be a worker.

So I run the start command, I can see it have the host IP address...

```
root@stimsonmt:/vllm-workspace# ray start --head --port=6379 --memory 24000000000
[...snip...]
Local node IP: 192.168.50.199
```

Let's try to join a node... I'm just giving it 12 out of 16 gigs because `yoda` is a workstation, so my GUI is hogging VRAM.

```
ray start --memory 12000000000 --address='192.168.50.199:6379'
```

Cause for a `w00t`, I can see the combination with `ray status` on the head (or the node, too)

```
root@stimsonmt:/vllm-workspace# ray status
2025-05-09 05:39:18,663 - INFO - NumExpr defaulting to 16 threads.
======== Autoscaler status: 2025-05-09 05:39:16.967610 ========
Node status
---------------------------------------------------------------
Active:
 1 node_a9d14d6a94d5e77fbdafdb8802dada067a37417f2f7ad98426aa3eb7
 1 node_4122ac6d486c994414de5ed9704c329002b5ec596fb38dd0806fe1fd
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/40.0 CPU
 0.0/2.0 GPU
 0B/33.53GiB memory
 0B/46.26GiB object_store_memory

Total Constraints:
 (no request_resources() constraints)
Total Demands:
 (no resource demands)
```

I compile the whole commands...

For the head:

```bash
export GLOO_DEBUG=2
export VLLM_LOGGING_LEVEL=DEBUG
export NCCL_SOCKET_IFNAME=enp45s0
export GLOO_SOCKET_IFNAME=enp45s0
export NCCL_ASYNC_ERROR_HANDLING=1
export RAY_memory_monitor_refresh_ms=0

ray start --head --port=6379 --memory 24000000000
```

For the node:microsoft/Phi-3.5-mini-instruct

```bash
export GLOO_DEBUG=2
export VLLM_LOGGING_LEVEL=DEBUG
export NCCL_SOCKET_IFNAME=eno1
export GLOO_SOCKET_IFNAME=eno1
export NCCL_ASYNC_ERROR_HANDLING=1
export RAY_memory_monitor_refresh_ms=0

ray start --memory 12000000000 --address='192.168.50.199:6379'
```

In prod, we'd see this as a service discovery challenge too, but, here, we're OK with some static IPs.

Now, let's see what Bijan's `vllm serve` command looks like:

```bash
vllm serve microsoft/Phi-3.5-mini-instruct \
  --tensor-parallel-size 2 \
  --pipeline-parallel-size 1 \
  --gpu-memory-utilization 0.8 \
  --max-model-len 4096 \
  --host 0.0.0.0 \
  --trust-remote-code
```

I had to fiddle with it, because that's a 2 node / 4 GPU setup, and it took me a few tries, but, I wound up getting this warning which I really like in my own way:

```
WARNING 05-09 06:03:18 [ray_utils.py:199] tensor_parallel_size=2 is bigger than a reserved number of GPUs (1 GPUs) in a node d97d078c6e6c301dafb330684dd1aede943dcbe95dabc32bfc3fa5c7. Tensor parallel workers can be spread out to 2+ nodes which can degrade the performance unless you have fast interconnect across nodes, like Infiniband. To resolve this issue, make sure you have more than 2 GPUs available at each node.
```

It mentions *Infiniband* -- which is the kind of thing I see get a lot of use in HPC networking scenarios, awesome. I don't have that setup, that's yet another blog article! But, after trying a number of combos, and failing, I think that warning is actually good news in our case.

For my test, I wound up choosing the same model Bijan (the YT creator) used, he's aiming for something small and simple, and that works for me...

Now, we just run the serve command on the `head`, and let it fire up....

We can see that we get a process loaded on the node, `yoda`.

```bash
root@yoda:/vllm-workspace# nvidia-smi \
  --query-compute-apps=pid,process_name,used_gpu_memory \
  --format=csv,noheader,nounits
2701, ray::RayWorkerWrapper.__ray_call__, 12104
```

And then I tried 

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "microsoft/Phi-3.5-mini-instruct",
    "prompt": "Name two former or current US senators from Vermont.",
    "max_tokens": 150,
    "temperature": 0.7
  }'

```

This calls for a: `wewwwwwwwwwwwwwwt`! There's a successful completion!

```
doug@stimsonmt:/tmp/testgrammar$ curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "microsoft/Phi-3.5-mini-instruct",
    "prompt": "Name two former or current US senators from Vermont.",
    "max_tokens": 150,
    "temperature": 0.7
  }'
{"id":"cmpl-8606a78a9d6b4a0ba7d8290c0d8e6893","object":"text_completion","created":1746797459,"model":"microsoft/Phi-3.5-mini-instruct","choices":[{"index":0,"text":"\n\nVermont, a state known for its picturesque landscapes and progressive political stance, has had several distinguished individuals serve in the United States Senate. Here are two former and two current US senators from Vermont:\n\nFormer Senators:\n\n1. Patrick Leahy: Serving since 1975, Patrick Leahy is a Democrat who has been one of the longest-serving senators in the history of the United States. He was re-elected multiple times and has had a significant impact on national policy and legislation.\n\n2. Jim Jeffords: He served from 1989 to 2007 as a Republican senator","logprobs":null,"finish_reason":"length","stop_reason":null,"prompt_logprobs":null}],"usage":{"prompt_tokens":12,"total_tokens":162,"completion_tokens":150,"prompt_tokens_details":null}}
```

Why it somehow mentioned Jeffords before Bernie is.... An absolute mystery to me, that seems so unlikely! But... Alas, it's also correct. And... Hurray!

And guess what? It also can use the structured output!

```
doug@stimsonmt:/tmp/testgrammar$  python test.py "shredding a quick uphill lap at Bolton after work"
rad
```

I'm also trying to figure out how to contribute some of this learning back to the codebase! Thankfully Russell also pointed me at [the vllm ci infra project](https://github.com/vllm-project/ci-infra). And I even joined [the vLLM slack](https://slack.vllm.ai/)!