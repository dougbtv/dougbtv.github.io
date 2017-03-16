---
author: dougbtv
comments: true
date: 2017-03-16 08:50:03-05:00
layout: post
slug: ansible-container
title: A (happy happy joy joy) ansible-container hello world!
category: nfvpe
---

Today we're going to explore [ansible-container](https://github.com/ansible/ansible-container), a project that gives you Ansible workflow for Docker. It provides a method of managing container images using ansible commands (so you can avoid a bunch of dirty bash-y Dockerfiles), and then provides a specification of "services" which is eerily similar (on purpose) to docker-compose. It also has paths forward for managing the instances of these containers on Kubernetes & OpenShift -- that's pretty tight. We'll build two images "ren" and "stimpy", which contain nginx and output some Ren & Stimpy quotes so we can get a grip on how it's all put together. It's [better than bad -- it's good](https://www.youtube.com/watch?v=-fQGPZTECYs)!

These steps were generally learned from dually the [ansible-container demo github page](https://ansible.github.io/ansible-container-demo/) and from the [getting started guide](http://docs.ansible.com/ansible-container/getting_started.html). It also leverages this [github project with demo ansible-container files](https://github.com/dougbtv/demo-ansible-container) I created which has all the files you need so you don't have to baby them all in an editor.

My editorial is that... This is really a great project. However, I don't consider it the be-all-end-all. I think it has an awesome purpose in the context of a larger ansible project. It's squeaky clean when you use it that way. Except for the directory structure which I find a little awkward. Maybe I'm doing that part slightly wrong, it's not terrible. I also think that Dockerfiles have their place. I like them, and in terms of some simpler apps (think, a Go binary) ansible-container is overkill, and your run of the mill pod spec when using k8s, raw and unabstracted isn't so bad to deal with -- in fact, it may be confusing in some places to abstract that. So, choose the right tool for the job is my advice. A-And I'd like a bike, and a Betsy Wetsherself doll, and a Cheesy-Bake Oven, and a Pulpy The Pup doll, and a gajillion green army men.

Ok, enough editorializing -- let's get some space madness and move onto getting this show on the road!

## Requirements

Fairly simple -- as per usual, we're using a CentOS 7.3 based virtual machine to run all these on. Feel free to use your workstation, but, I put this all in a VM so I could isolate it, and see what you needed given a stock CentOS 7.3 install. Just as a note, my install is from a CentOS 7.3 generic cloud, and the basics are based on that.

Also -- you need a half dozen gigs free of disk, and a minimum of 4 gigs of memory. I had a 2 gig memory VM and it was toast (ahem, powdered toast) when I went to do the image builds, so, keep that in mind.

Since I have a throw-away VM, I did all this as root, you can be a good guy and use sudo if you please.

## Install Docker!

Yep, you're going to need a docker install. We'll just use the latest docker from CentOS repos, that's all we need for now.

```
yum install -y docker
systemctl enable docker
systemctl start docker
docker images
docker -v
```

## Install ansible-container

We're going to install ansible-container from source so that we can have the `0.3.0` version (because we want the docker-compose v2 specification)

Now, go and install the every day stuff you need (like, your editor). I also installed tree so I could use it for the output here. Oh yeah, and you need git!

So we're going to need to update some python-ish things, especially epel to get python-pip, then update python-pip, then upgrade setuptools.

```
yum install -y epel-release
yum install -y python-pip git
pip install --upgrade pip
pip install -U setuptools
```

Now, let's clone the project and install it. These steps were generally learned from [these official installation directions](https://github.com/ansible/ansible-container#installing).

```
git clone https://github.com/ansible/ansible-container.git
cd ansible-container/
python ./setup.py install
```

And now you should have a version 0.3.0-ish ansible-container.

```
[user@host ansible-container]$ ansible-container version
Ansible Container, version 0.3.0-pre
```

## Let's look at `ansible-container init`

In a minute here we're going to get into using a full-out project, but, typically when you start a project, there's a few things you're going to do. 

1. You're going to use `ansible-container init` to scaffold the pieces you need.
2. You'll use `ansible-container install some.project` to install ansible galaxy modules into your project.

So let's give that a test drive before we go onto our custom project.

Firstly, make a directory to put this in as we're going to throw it out in a minute.

```
[user@host ~]$ cd ~
[user@host ~]$ mkdir foo
[user@host ~]$ cd foo/
[user@host foo]$ ansible-container init
Ansible Container initialized.
```

Alright, now you can see it created a `./ansible/` directory there, and it has a number of files therein. 

## Installing ansible galaxy modules for ansible-container

Now let's say we're going to install an nginx module from ansible galaxy. We'd do it like so...

```
[user@host foo]$ ansible-container install j00bar.nginx-container
```

Note that this will take a while the first time because it's pulling some ansible-specific images. 

Once it's done with the pull, let's inspect what's there.

## Inspecting what's there.

Let's take a look at what it looks like with `ansible-container init` and then an installed role.

```
[user@host foo]$ tree
.
└── ansible
    ├── ansible.cfg
    ├── container.yml
    ├── main.yml
    ├── meta.yml
    ├── requirements.txt
    └── requirements.yml

1 directory, 6 files
```

Here's what each file does.

* `container.yml` this is a combo of both inventory and "docker-compose"
* `main.yml` this is your main playbook which runs plays against the defined containers in the container.yml
* `requirements.{txt,yml}` is your python & role deps respectively
* `meta.yml` is for ansible galaxy (should you publish there).
* `ansible.cfg` is your... ansible config.

## Let's make our own custom playbooks & roles!

Alright, so go ahead and move back to home and clone my [demo-ansible-container](https://github.com/dougbtv/demo-ansible-container) repo. 

The job of this role is to create two nginx instances (in containers, naturally) that each serve their own custom HTML file (it's more like a text file). 

So let's clone it and inspect a few things.

```
$ cd ~
$ git clone https://github.com/dougbtv/demo-ansible-container.git
$ cd demo-ansible-container/
```

## Inspecting the project

Now that we're in there, let's show the whole directory structure, it's basically the same as earlier when we did `ansible-container init` (as I started that way) plus it adds a `./ansible/roles/` directory which contains roles just as you'd have in your run-of-the-mill ansible project.

```
.
├── ansible
│   ├── ansible.cfg
│   ├── container.yml
│   ├── main.yml
│   ├── meta.yml
│   ├── requirements.txt
│   ├── requirements.yml
│   └── roles
│       ├── nginx-install
│       │   └── tasks
│       │       └── main.yml
│       ├── ren-html
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       │       └── index.html
│       └── stimpy-html
│           ├── tasks
│           │   └── main.yml
│           └── templates
│               └── index.html
└── README.md
```

You'll note there's everything we had before, plus three roles.

* `nginx-install`: which install (and generally configures) nginx
* `ren-html` & `stimpy-html`: which places specific HTML files in each container

Now, let's look specifically at the most imporant pieces.

First, our `container.yml`

```
[user@host demo-ansible-container]$ cat ansible/container.yml 
version: '2'
services:
  ren:
    image: centos:centos7
    ports:
      - "8080:80"
    # user: nginx
    command: "nginx" # [nginx, -c, /etc/nginx/nginx.conf]
    dev_overrides:
      ports: []
      command: bin/false
    options:
      kube:
        runAsUser: 997
        replicas: 2
      openshift:
        replicas: 3
  stimpy:
    image: centos:centos7
    ports:
      - "8081:80"
    # user: nginx
    command: [nginx, -c, /etc/nginx/nginx.conf]
    dev_overrides:
      ports: []
      command: bin/false
    options:
      kube:
        runAsUser: 997
        replicas: 2
      openshift:
        replicas: 3
registries: {}
```

Whoa whoa, whoa Doug! There's too much there. Yeah, there kind of is. I also put in some goodies to tempt you to look further ;) So, you'll notice this looks very very much like a docker-compose yaml file. 

Mostly though for now, looking at the `services` section, there's two services listed `ren` & `stimpy`. 

These comprise the inventory we'll be using. And they specify things like... What ports we're going to run the containers on, especially we'll be using ports `8080` and `8081` which both map to port 80 inside the container.

Those are the most important for now.

So let's move onto looking at the `main.yml`. This is sort of your site playbook for all your containers.

```
[user@host demo-ansible-container]$ cat ansible/main.yml 
- hosts: all
  gather_facts: false
- hosts: 
    - ren
    - stimpy
  roles:
    - role: nginx-install
- hosts: ren
  roles:
    - role: ren-html
- hosts: stimpy
  roles:
    - role: stimpy-html
```

So, looks like any other ansible playbook, awesome! The gist is that we use the "host" names `ren` & `stimpy` and we run roles against them.

You'll see that both ren & stimpy have nginx installed into them, but, then use a specific role to install some HTML into each container image. 

Feel free to deeply inspect the roles if you so please, they're simple.

## Onto the build!

Now that we've got that all setup, we can go ahead and build these container images.

Let's do that now. Make sure you're in the `~/demo-ansible-container` working dir and not the `~/demo-ansible-container/ansible` dir or this won't work (one of my pet peeves with ansible-container, tbh)

```
[user@host demo-ansible-container]$ ansible-container build
```

You'll see that it spins up some containers and then runs those plays, and you can see it having some specificity to each "host" (each container, really). 

When it's finished it will go and commit the images, to save the results of what it did to the images. 

Let's look at the results of what it did.

```
[user@host demo-ansible-container]$ docker images
REPOSITORY                                    TAG                 IMAGE ID            CREATED             SIZE
demo-ansible-container-ren                    20170316142619      ba5b90f9476e        5 seconds ago       353.9 MB
demo-ansible-container-ren                    latest              ba5b90f9476e        5 seconds ago       353.9 MB
demo-ansible-container-stimpy                 20170316142619      2b86e0872fa7        12 seconds ago      353.9 MB
demo-ansible-container-stimpy                 latest              2b86e0872fa7        12 seconds ago      353.9 MB
docker.io/centos                              centos7             98d35105a391        16 hours ago        192.5 MB
docker.io/ansible/ansible-container-builder   0.3                 b606876a2eaf        12 weeks ago        808.7 MB
```

As you can see it's got it's special `ansible-container-builder` image which it uses to bootstrap our images. 

Then we've got our `demo-ansible-container-ren` and `demo-ansible-container-stimpy` each with two tags. One for `latest` and then anotehr tag with the date and time.

## And we run it.

Ok, everything's built, let's run it.

```
ansible-container run --detached --production
```

You can run without --production and it will just run `/bin/false` in the container, which may be confusing, but, it's basically a no-operation and you could use it to inspect the containers in development if you wanted.

When that completes, you should see two containers running.

```
[user@host demo-ansible-container]$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS                  NAMES
7b160322dc26        demo-ansible-container-ren:latest      "nginx"                  27 seconds ago      Up 24 seconds       0.0.0.0:8080->80/tcp   ansible_ren_1
a2f8dabe8a6f        demo-ansible-container-stimpy:latest   "nginx -c /etc/nginx/"   27 seconds ago      Up 24 seconds       0.0.0.0:8081->80/tcp   ansible_stimpy_1
```

Great! Two containers up running on ports 8080 & 8081, just as we wanted.

## Finally, verify the results.

You can now see you've got Ren & Stimpy running, let's see what they have to say.

```
[user@host demo-ansible-container]$ curl localhost:8080
You fat, bloated eeeeediot!

[user@host demo-ansible-container]$ curl localhost:8081
Happy Happy Happy! Joy Joy Joy!
```

And there we go, two images built, two containers running, all with ansible instructions on how they're built!

Makes for a very nice paradigm to create images & spin up containers in the context of an Ansible project.