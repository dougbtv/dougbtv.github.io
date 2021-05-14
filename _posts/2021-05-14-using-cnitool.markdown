---
author: dougbtv
comments: true
date: 2021-05-14 08:00:00-05:00
layout: post
slug: using-cnitool
title: cnitool -- your CNI Swiss Army knife
category: nfvpe
---

If you're looking at developing (or debugging!) CNI plugins, you're going to need a workflow for developing CNI plugins -- something that really lets you get in there, and see exactly what a CNI plugin is doing. You're going to need a bit of a swiss army knife, or something that [slices, dices, and makes juilienne fries](https://en.wikipedia.org/wiki/Veg-O-Matic). `cnitool` is just the thing to do the job. Today we'll walk through setting up `cnitool`, and then we'll make a "dummy" CNI plugin to use it with, and we'll run a reference CNI plugin.

We'll also cover some of the basics of the information that's passed to and from the CNI plugins and CNI itself, and how you might interact with that information, and how you might inspect a container that's been plumbed with interfaces as created by a CNI plugin. 

In this article, we'll do this entirely without interacting with Kubernetes (and save it for another time!). And we actually do it without a container runtime at all -- no docker, no crio. We just create the network namespace by hand. But the same kind of principles apply with both a container runtime (docker, crio) or a container orchestration enginer (e.g. k8s)

You might remember my [blog article about a workflow for developing CNI plugins](http://dougbtv.com/nfvpe/2017/06/22/cni-tutorial/). That article uses the [docker-run.sh](https://github.com/containernetworking/cni/blob/master/scripts/docker-run.sh), which is still totally valid. You might look at it for a reference, but CNI tool gives a bit more granularity.

# Prerequisites

* Golang installed and configured on your system.
* I used a Fedora environment, these steps probably work elsewhere.

# Setting up `cnitool` and the reference CNI plugins.

Basically, all the steps necessary to install cnitool are available in [the cnitool README](https://github.com/containernetworking/cni/tree/master/cnitool). I'll summarize them here, but, it may be worth a reference.

Install cnitool...

```
go get github.com/containernetworking/cni
go install github.com/containernetworking/cni/cnitool
```

You can test if it's in your path and operational with:

```
cnitool --help
```

Next, we'll compile the "reference CNI plugins" -- these are a series of plugins that are offered by the CNI maintainers that create network interfaces for pods (as well as provide a number of "meta" type plugins that alter the properties, attributes, and what not of a particular container's network). We also set our `CNI_PATH` variable (which is used by cnitool to know where these plugin executables are)

```
git clone https://github.com/containernetworking/plugins.git
cd plugins
./build_linux.sh
export CNI_PATH=$(pwd)/bin
echo $CNI_PATH
```

Alright, you're basically all setup at this point.

# Creating a netns and running cnitool against it

We'll need to create a CNI configuration. For testing purposes, we're going to create a configuration for the bridge CNI.

Create a directory and file at `/tmp/cniconfig/10-myptp.conf` with these contents:

```
{
  "cniVersion": "0.4.0",
  "name": "myptp",
  "type": "ptp",
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "172.16.29.0/24",
    "routes": [{
      "dst": "0.0.0.0/0"
    }]
  }
}
```

And then set your CNI configuration directory by exporting this variable as:

```
export NETCONFPATH=/tmp/cniconfig/
```

First we create a netns -- a network namespace. This is kind of a privately sorta-jailed space in which network components live, and is the basis of networking in containers, "here's your private namespace in which to do your network-y things". This, from a CNI point of view, is equivalent to the "sandbox" which is the basis container of pods that run in kubernetes. In k8s we'd have one or more containers running inside this sandbox, and they'd share the networks as in this network namespace.

```
sudo ip netns add myplayground
```

You can go and list them to see that it's there...

```
sudo ip netns list | grep myplayground
```

Now we're going to run `cnitool` with `sudo` so it has the appropriate permissions, and we're going to need to pass it along our environment variables and our path to cnitool (if your root user doesn't have a go environment, or isn't configured that way), for me it looks like:

```
sudo NETCONFPATH=$(echo $NETCONFPATH) CNI_PATH=$(echo $CNI_PATH) $(which cnitool) add myptp /var/run/netns/myplayground
```

Let's breakdown what this is doing more or less...

* `NETCONFPATH=$(echo $NETCONFPATH) CNI_PATH=$(echo $CNI_PATH)` sets our environment variables to tell tool
* `$(which cnitool)` figures out the path of `cnitool` so that inside your sudo environment, you don't need your GOPATH (you're rad if you have that setup, though)
* `add myptp /var/run/netns/myplayground` says that `add` is the CNI method which is being invoked, `myptp` is our configuration, and the `/var/run/...` is the path to the netns that we created.

You should get some output that looks like:

```
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "veth20b2acac",
            "mac": "62:22:15:72:b2:29"
        },
        {
            "name": "eth0",
            "mac": "42:48:16:0b:e9:98",
            "sandbox": "/var/run/netns/myplayground"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 1,
            "address": "172.16.29.3/24",
            "gateway": "172.16.29.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
}
```

You can then actually do a ping out that interface, with:

```
sudo ip -n myplayground addr
sudo ip netns exec myplayground ping -c 1 4.2.2.2
```

And you can use nsenter to more interactively play with it, too...

```
sudo nsenter --net=/var/run/netns/myplayground /bin/bash
[root@host dir]# ip a
[root@host dir]# ip route
[root@host dir]# ping -c 5 4.2.2.2
```

# Let's interactively look at a CNI plugin running with cnitool.

What we're going to do is create a shell script that is a CNI plugin. You see, CNI plugins can be executables of any variety -- they just need to be able to read from [stdin, and write to stdout and stderr](https://www.howtogeek.com/435903/what-are-stdin-stdout-and-stderr-on-linux/).

This is kind of a blank slate for a CNI plugin that's made with bash. You could use this approach, but, in reality -- you'll probably write these applications with go. Why? Well, especially because there's the CNI libraries ([especially libcni](https://github.com/containernetworking/cni/tree/master/libcni)) which you would use to be able to express some of these ideas about CNI in a more elegant fashion. Take a look at how Multus uses CNI's `skel` (skeletal components, for the framework of your CNI plugin) in its main routine to call the methods as CNI has called them. Just read through [Multus' main.go](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/cmd/main.go) and look how it [imports skel](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/cmd/main.go#L26) and then using skel [calls our method to add](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/cmd/main.go#L47-L48) when CNI ADD is used.

First, let's make a cni configuration for our dummy plugin. I made mine at `/tmp/cniconfig/05-dummy.conf`.

```
{
  "cniVersion": "0.4.0",
  "name": "mydummy",
  "type": "dummy"
}
```

There's not a lot to pay attention to here, the most important things are:

* the `type` field which must have the same name as our executable on disk -- which are both going to be `dummy`
* the `name` field is the name we'll reference in our `cnitool` command, which will be `mydummy`.

Now, in the path where we have our reference CNI plugins, lets add another file, name it `dummy`, and then make sure its executable. In my case I did a:

```
vi ./bin/dummy
chmod 0755 ./bin/dummy
```

I made mine with [the contents from this gist](https://gist.github.com/dougbtv/294b9599d897be55a97396e03cba3dae).

The first thing to note is that the majority of this file is to actually just setup some logging for looking at the CNI parameters, and all the magic happens in the last 3-4 lines.

Mainly, we want to [output 3 environment using these three lines](https://gist.github.com/dougbtv/294b9599d897be55a97396e03cba3dae#file-dummy-sh-L58-L60). These are some environment variables that are sent to us from CNI and that a CNI plugin can use to figure out the netns, the container id, and the CNI command.

Importantly -- since [we have this DEBUG variable turned on](https://gist.github.com/dougbtv/294b9599d897be55a97396e03cba3dae#file-dummy-sh-L3), we're outputting via stderr... if there's any stderr output during a CNI plugin run, this is considered a failure, as that's what you're supposed to do when you error out, is output to stderr.

And last but not least, we [output a CNI result at the bottom line](https://gist.github.com/dougbtv/294b9599d897be55a97396e03cba3dae#file-dummy-sh-L62), which calls [this function which outputs a (sorta kinda realistic) CNI result](https://gist.github.com/dougbtv/294b9599d897be55a97396e03cba3dae#file-dummy-sh-L29-L41).

You can turn that off, but we have it on for demonstrative purposes so you can easily see the what those variables are.

So, let's run it!

```
sudo NETCONFPATH=$(echo $NETCONFPATH) CNI_PATH=$(echo $CNI_PATH) $(which cnitool) add mydummy /var/run/netns/dummyplayground
```

And you can see output that looks like:

```
CNI method: ADD
CNI container id: cnitool-06764c511c35893f831e
CNI netns: /var/run/netns/dummyplayground
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "dummy"
        }
    ],
    "dns": {}
}
```

Here we'll see that there's a lot of information that we as humans already know, since we're executing CNI tool, but it demonstrates how a CNI plugin interacts with this information, it's telling us that it:

* Knows that we're doing a CNI `ADD` operation.
* We're using a netns that's called `dummyplayground`
* It's outputting a CNI result.

These are the general basics of what a CNI plugin needs in order to operate. And then... from there, the sky's the limit. A more realistic plugin might

And to learn a bit more, you might think about looking at some of the reference CNI plugins, and see what they do to create interfaces inside these network namespaces.

# But what if my CNI plugins interacts with Kubernetes!?

...And that's for next time! You'll need a Kubernetes environment of some sort.