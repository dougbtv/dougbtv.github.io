---
author: dougbtv
comments: true
date: 2017-06-22 17:30:01-05:00
layout: post
slug: cni-tutorial
title: Let's create a workflow for writing CNI plugins (including writing your first CNI plugin!)
category: nfvpe
---

In this tutorial, we're going to write a [CNI](https://github.com/containernetworking/cni) plugin, that is a "container network interface" plugin, that in this case we'll specifically use in Kubernetes. A CNI plugin executes on start & stop of a container, and you use it to, generally, modify the infra container's network namespace in order to configure networking for the pod. We can use this to customize how we setup networking. Today, we'll both write a simple Go application to say "Hello, world!" to CNI to inspect how it works a little bit, and we'll follow that up by looking at my CNI plugin [Ratchet CNI](https://github.com/dougbtv/ratchet-cni) (an implementation of [koko](https://github.com/redhat-nfvpe/koko) in CNI) a little bit to grok the development workflow.

Our goal today is to:

* Run a "dummy" CNI plugin of our own build, to show some of the moving parts
* And run Ratchet CNI -- to introduce some of the work-flow that I used to build it.

A lot of what's here borrows heavily from [the running the plugins section of the CNI readme](https://github.com/containernetworking/cni#running-the-plugins). We'll add on to here by introducing some key concepts, and get you started in writing your own plugin.

## Requirements

While it's not required -- you probably want to have a Kubernetes environment setup for yourself where you can experiment with deploying the plugins in Kubernetes proper. In my case, I used a Kubernetes master to check out my stuff "in development" and then also used a simple cluster with a master and single minion. If you need a Kubernetes lab environment, maybe I could tempt you to [try using my lab playbooks](dougbtv.com//nfvpe/2017/02/16/kubernetes-1.5-centos/). I also tend to assume a CentOS environment. I don't use Kubernetes itself during this tutorial, but, you'll certainly level up faster if you take the steps here and implement some of these ideas on Kube as a DIY exercise.

You can get away without golang if you just go up to the point where we create a dummy plugin. If you want to go further, you'll need golang, and preferably Ansible to go ahead with running and inspecting Ratchet CNI.

On whatever box you use as I use my master, you're going to need to install golang, e.g. on CentOS `yum install -y golang`, and you'll need Docker (unless you're cool enough to have another container runtime, in which case I salute you and you can go ahead with adapting towards that). 

Also -- you might want to think about installing an even later version of Golang from [go-repo.io](http://go-repo.io/).

Lastly, you might see some mix here between a prompt as an unprivileged user, and root. The best case scenario is that you [setup a regular user to use Docker](https://askubuntu.com/questions/477551/how-can-i-use-docker-without-sudo)... or you can just use root.

## Some basics behind CNI.

When Kubernetes starts up your pod (a logical group of containers), something it will do is create a "infra container" -- this is a container that has a shared network namespace (among other namespaces) that all the containers in the pod share.

This means that any networking elements you create in that infra container will be available to all the containers in the pod. This also means that as containers come and go within that pod, the networking stays stable. 

If you have a running Kubernetes (which has some pods running), you can perform a `docker ps` and see containers that often running with `gcr.io/google_containers/pause-amd64` image, and they're running a command that looks like `/pause`. If you're running OpenShift, the same concept applies, but, it may be a different image and command. In theory this is a lightweight enough container that it "shouldn't really die" and should be available to all containers within that pod.

As Kubernetes creates this infra container, it also will call an executable as specified in the `/etc/cni/net.d/*conf` files. Kubernetes passes the contents of this 

Kubernetes then uses the same config and calls the same binary when the pod is destroyed, too.

If you want even more detail, you can checkout the [CNI specification itself](https://github.com/containernetworking/cni/blob/master/SPEC.md).


## Setting up your environment.

First thing we'll do is clone the [CNI](https://github.com/containernetworking/cni) repo proper, e.g.:

```
git clone https://github.com/containernetworking/cni.git
```

If you're not running in a Kubernetes environment, you'll also need to build some [plugins](https://github.com/containernetworking/plugins). We'll clone another repo in order to do so. You can build them up with a recipe like:

```
git clone https://github.com/containernetworking/plugins.git
cd plugins
./build.sh
cp ./bin/* /opt/cni/bin
```

Then you can copy those binaries out to wherever you need. In my case, since I already have a running Kubernetes environment, I'm assuming you have binaries in `/opt/cni/bin`.

Last but not least, you're going to need jq -- as the scripts we're using coming up require it.

```
[centos@cni ~]$ sudo curl -Ls -o /usr/bin/jq -w %{url_effective} https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64
[centos@cni ~]$ sudo chmod +x /usr/bin/jq
[centos@cni ~]$ /usr/bin/jq  --version
jq-1.5
```

## Using the handy-dandy docker-run.sh

In the clone of `containernetworking/cni` -- you'll find a `./scripts` directory which has a `docker-run.sh` this is a wrapper around the docker run command that invokes docker in such a way as to have a 

Before we run those, we're going to want to set the path of our CNI executables, and additionally where our configs live.

```
[root@cni scripts]# export CNI_PATH=/opt/cni/bin/
[root@cni scripts]# export NETCONFPATH=/etc/cni/net.d
```

Now that we have those, we're going to create a simple CNI configuration, and we'll run one of the default plugins.

We'll shamelessly borrow the two configs from the official CNI readme, which include using the `bridge` type plugin, and a loopback. You'll notice that these configs are "just JSON"

```
$ mkdir -p /etc/cni/net.d
$ cat >/etc/cni/net.d/10-mynet.conf <<EOF
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF
$ cat >/etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.2.0",
    "type": "loopback"
}
EOF
```

With those in place, we can now run a container. Let's go for it.

```
[root@kube-mult-master scripts]# ./docker-run.sh --rm busybox ifconfig | grep -Pi "(eth0|lo|inet addr)"
eth0      Link encap:Ethernet  HWaddr 0A:58:0A:16:00:03  
          inet addr:10.22.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
```

You can see that we have the two pieces we specified, a loopback, and a bridge to cni0 with a `10.22.0.0/16`

Now that you're done with that, let's delete those two configs.

```
[root@kube-mult-master scripts]# rm /etc/cni/net.d/*conf
```

## Let's make our own dummy plugin!

Cool, so now that we have that... We're going to make a new config, and we'll create a "dumb" bash script that we'll have execute.

```
cat >/etc/cni/net.d/10-mynet.conf <<EOF
{
    "cniVersion": "0.2.0",
    "name": "my_dummy_network",
    "type": "dummy"
}
EOF
```

Now we can create our dummy script.

```
cat >/opt/cni/bin/dummy <<EOF
#!/bin/bash
logit () {
 >&2 echo \$1
}

logit "CNI method: \$CNI_COMMAND"
logit "CNI container id: \$CNI_CONTAINERID"
logit "-------------- Begin config"
while read line
do
  logit "\$line"
done < /dev/stdin
logit "-------------- End config"
EOF
```

And then give it proper permissions, to make it executable:

```
[root@kube-mult-master scripts]# chmod 0755 /opt/cni/bin/dummy
```

Now that it's in place, let's look at a few things in this script, as it's going to tell us a few key bits of information we're going to find helpful as we go along to create real CNI plugins.

Firstly: Anything that's written to `stderr` is going to appear when we use the `docker-run.sh` utility. That's why we have the `logit()` function that does something like `>&2 echo "foo"` as that writes to `stderr`. This is really handy for debugging. Note that when you use it in kubernetes, it won't show you anything, so if you need to debug there you'll have to create some other facility for logging.

Next -- you'll notice there's two ways that information is passed to your CNI plugin.

* Environment variables.
* Config file via `stdin`.

The list of environment variables are available [in the CNI spec in the Overview section](https://github.com/containernetworking/cni/blob/master/SPEC.md#overview-1) (down towards the bottom of that section). 

You'll notice that there's a part of the script that reads:

```
logit "CNI method: $CNI_COMMAND"
```

This tells us if it's on creation or deletion of the pod, and will come up as either `ADD` or `DEL`.

Then there's a section where we read from stdin.

```
logit "-------------- Begin config"
while read line
do
  logit "\$line"
done < /dev/stdin
logit "-------------- End config"
```

The whole config file is then passed in here. So, Kubernetes (or this handy `docker-run.sh`) has already read this and knows what plugin to run, and then... It knows how to send it all to us.

In your plugin itself, you'll then read this in to read any options that want to add.

If you want some more information that's purely in Go code, take a look at [the skel go modules in the CNI repo](https://github.com/containernetworking/cni/blob/master/pkg/skel/skel.go). This shows you exactly what CNI is doing to pass some information around.

Alright, enough jibber jabber -- I want to run this dummy plugin already! Here you go:

```
[root@kube-mult-master scripts]# ./docker-run.sh --rm busybox ifconfig 
CNI method: ADD
CNI container id: d416ce8dc911a91b080530e1d18e033637a736c0affc707af5f219c59e919672
-------------- Begin config
{
"cniVersion": "0.2.0",
"name": "my_dummy_network",
"type": "dummy"
}
-------------- End config
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

CNI method: DEL
CNI container id: d416ce8dc911a91b080530e1d18e033637a736c0affc707af5f219c59e919672
-------------- Begin config
{
"cniVersion": "0.2.0",
"name": "my_dummy_network",
"type": "dummy"
}
-------------- End config
```

In this case, you'll see the output from the dummy plugin both before and after the `ifconfig` output, as we run the container with the `docker-run.sh` script, and it invokes our plugin both on `ADD` and on `DEL`

Congratulations -- you have officially written a CNI plugin now. It's not much (seeing that it, well doesn't create any networking), but, it demonstrates what the moving pieces are to get an application to run.

## Let's inspect a "more real" plugin.

So -- chances are, you're not actually going to write a plugin in bash. Or, I hope you don't / I don't wish that on my worst enemy. You're probably going to use Go. Not because it's better or worse than anything else -- but because you're entering into a world of [Gophers](https://blog.golang.org/gopher). And there's lots of utilities out there for interacting with CNI itself, and with the containers -- something we'll likely do a lot as we write CNI plugins.

So why the quotes on "more real" plugin? Because we're looking at [Ratchet CNI](https://github.com/dougbtv/ratchet-cni), and it's primarily an experiment, which leverages a more powerful technology -- [koko](https://github.com/redhat-nfvpe/koko), a way of connecting containers with veth or vxlan to provide some network isolation for containers (and maybe some service function chaining, later on).

The Ratchet CNI is primarily a wrapper that can invoke koko. It does do some interesting things, but, the most interesting part of CNI, well... Is the networking! So, maybe it's fair case.

## Looking at some important bits in Ratchet

Let's look at some of the important bits in [Ratchet CNI](https://github.com/dougbtv/ratchet-cni), starting with the dependencies. The primary script we're going to look at is `./ratchet/ratchet.go` -- which is what we compile down and is the main terminal binary that gets run by CNI. There's more to how ratchet is designed, but, for today since we're looking at build your own first CNI plugin -- we'll stick to the most interesting stuff there.

### The Ratchet dependencies

Some of the most important dependencies are [in these lines in ratchet.go](https://github.com/dougbtv/ratchet-cni/blob/da4fb04078a05b012a747ed545874ae933a8f950/ratchet/ratchet.go#L33-L38), which includes:

*  `github.com/containernetworking/cni/pkg/skel`: Skeleton for CNI to read `stdin` & environment variables.
*  `github.com/containernetworking/cni/pkg/invoke`: Allows us to use the `DelegateAdd` method which we use to call other plugins.
*  `github.com/containernetworking/cni/pkg/types`: Some common types that are used by the CNI packages, including the `NetConf` type which defines our config JSON that we read from `stdin`.

There's also a Docker client that we use to pick some additional metadata from the pod.

### The `main` method

The `main()` method of the application is really just calling `skel` -- [as seen here](https://github.com/dougbtv/ratchet-cni/blob/master/ratchet/ratchet.go#L453), which looks like:

```
skel.PluginMain(cmdAdd, cmdDel)
```

So we let `skel` do some work for us -- it will call either of these methods (which are local to the Ratchet application), either `cmdAdd` or `cmdDel` (called on either creation or deletion of the pod). In those methods -- we're able to have a return from skel that includes the JSON config, which we can then parse and read to get some custom properties out of it.

## Running the Ratchet CNI playbooks

You might not actually care about what Ratchet itself is doing, but, what you may care about is how I setup my development environment and how I manage that so I can hack on Ratchet, and then run it.

I do all of the editing of the application in an IDE (Sublime Text, for me) on my workstation. Then I keep my workstation clean of running any of the dependencies of this application, because, in my opinion I should have a place where I can store how to create all of those dependencies -- which is why I choose an Ansible playbook to do that for me. I then use an Ansible playbook to create my environment where these will run (which is a small kubernetes cluster) and then I can both run against the quick-to-debug-against `docker-run.sh` -- and also deploy it to Kubernetes, for a final test.

While we're talking about -- you might also like taking a look at the `.travis.yml` file, too. Which shows you the exact steps that are taken in order to validate that the plugin is working -- and should in theory give you all the steps you need to get it working yourself.

### Using the Ansible playbooks

In the `./utils` directory there are some rudimentary Ansible playbooks. If you're going to use them, they do assume you have a Kubernetes master, and a Kubernetes minion (at least one). 

Go ahead and edit the `./utils/remote.inventory` file and change out the bits you want, especially the location of your boxen, and you might not need my `ansible_ssh_common_args` in the host variables (unless you're using my ansible playbooks for labs, in which case -- that might be handy)

After you've got that, there's two playbooks you're going to run...

* `./utils/sync-and-build.yml`: rsyncs code from local machine to remote master, and compiles it on the master -- then copies the binary to all the minions -- also templates the configs, it should be generally ready to use in Kubernetes at this point.
* `./utils/docker-run.yml`: Sets up everything to run with the `docker-run.sh` from the CNI repo.

So you'd run the two commands in series like so:

```
$ ansible-playbook -i remote.inventory sync-and-build.yml
$ ansible-playbook -i remote.inventory docker-run.yml
```

Now that you've got those two in hand. Now you have run the Ratchet CNI plugin! The real usefulness in the context of this article is to 

You can verify it by doing a `docker ps` and then validate the functionality of it (which is to provide some network isolation between containers using [koko](https://github.com/redhat-nfvpe/koko)). By doing something like...

```
[root@kube-mult-master scripts]# docker exec -it primary ifconfig | grep in1
in1       Link encap:Ethernet  HWaddr 1A:73:4A:78:B7:21  
[root@kube-mult-master scripts]# docker exec -it primary ping -c 1 192.168.2.101
PING 192.168.2.101 (192.168.2.101): 56 data bytes
64 bytes from 192.168.2.101: seq=0 ttl=64 time=0.086 ms

--- 192.168.2.101 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.086/0.086/0.086 ms
```






