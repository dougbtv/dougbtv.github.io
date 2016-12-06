---
author: dougbtv
comments: true
date: 2016-12-06 16:56:10-05:00
layout: post
slug: ansible-cira
title: Hello Ansible CIRA!
category: nfvpe
---

Today we're going to look at [CIRA](https://github.com/redhat-nfvpe/ansible-cira). CIRA is a tool to deploy a CI reference architecture to test OpenStack. I'm going to go with the Docker deployment option, as that's the environment that I tend towards. Today we'll get it up and running here.

----

## Requirements

In short we'll need these things:

* Ansible
* Docker
* Docker-compose
* Python shade module
* A clone of ansible cira
* The ansible-galaxy roles provided.

I've already got docker on my host, so let's just go ahead and install `docker-compose` which is a requirement

```
[root@undercloud stack]# curl -L "https://github.com/docker/compose/releases/download/1.9.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
[root@undercloud stack]# chmod 0755 /usr/local/bin/docker-compose
[root@undercloud stack]# docker-compose --version
```

We also need to install shade, in my case I already have it installed as it's a requirement for openshift-ansible when you're using the Openstack method of deploying that. Speaking of which you also need ansible, which I already had for the same requirement.

```
[stack@undercloud ~]$ pip install --user shade
```

Now we'll clone the repository

```
[stack@undercloud ~]$ git clone https://github.com/redhat-nfvpe/ansible-cira.git
[stack@undercloud ~]$ cd ansible-cira
```

And finally for the requirements make sure you've got the ansible galaxy roles

```
[stack@undercloud ansible-cira]$ ansible-galaxy install -r requirements.yml
```

## CIRA setup

We're going to:

* Create the `clouds.yml` user config file
* Initialize the custom ansible vars


We're going to need a `clouds.yml` file.

```
[stack@undercloud ~]$ mkdir ~/.config/openstack
[stack@undercloud ~]$ touch ~/.config/openstack/clouds.yml
```

Then let's reference our overcloudrc to get the things we need in here.

```
[stack@undercloud ~]$ cat overcloudrc 
```

And then I'll setup my `clouds.yml` file. Here's what mine winds up looking like...

```
[stack@undercloud ~]$ cat ~/.config/openstack/clouds.yml
clouds:
    mycloud:
        auth:
            auth_url: http://192.168.1.150:5000/v2.0
            username: admin
            password: fDZmuDw6U2pR29TYvTyfpytsM
            project_name: "Doug's OpenShift-on-Openstack"
```

We need to init some ansible vars, so make sure you're ready for this requirement by making a blank `cira_vars.yml` file.

```
[stack@undercloud ~]$ mkdir -p ~/.ansible/vars/
[stack@undercloud ~]$ touch ~/.ansible/vars/cira_vars.yml
```

I also preloaded my vars a little bit...

```
[stack@undercloud ansible-cira]$ cat ~/.ansible/vars/cira_vars.yml 
---
cloud_name_prefix: redhat                  # virtual machine name prefix
cloud_name: mycloud                        # same as specified in clouds.yml
```

We're also going to add a jenkins slave, as I think it's required. Err, first time I ran I got a `fatal: [jenkins_master]: FAILED! => {"failed": true, "msg": "'dict object' has no attribute 'jenkins_slave'"}`. For what it's worth.


We're going to use the undercloud itself as a slave. Unwise? Maybe.
```
[stack@undercloud ansible-cira]$ ssh-copy-id -i ~/.ssh/id_rsa.pub stack@127.0.0.1
```

I went ahead and altered the `hosts/containers` to add a slave.

```
[stack@undercloud ansible-cira]$ cat hosts/containers 
# This inventry file is for container (docker case)
# these names map to container name
jenkins_master
logstash
elasticsearch
kibana

[jenkins_slave]
slave01 ansible_connection=ssh ansible_host=127.0.0.1 ansible_user=stack

[jenkins_slave:vars]
slave_description=CIRA Testing Node
slave_remoteFS=/home/stack
slave_port=22
slave_credentialsId=stack-credential
slave_label=cira
```

## Start it up

Go ahead and run the docker-compose, to put the composure up in daemon mode.

```
[stack@undercloud ansible-cira]$ docker-compose up -d
[stack@undercloud ansible-cira]$ docker ps
CONTAINER ID        IMAGE                        COMMAND             CREATED             STATUS              PORTS               NAMES
f45e3b3e5f1d        ansiblecira_logstash         "/sbin/init"        13 seconds ago      Up 9 seconds                            logstash
3e076480315c        ansiblecira_kibana           "/sbin/init"        13 seconds ago      Up 10 seconds                           kibana
257ec8b685e4        ansiblecira_elasticsearch    "/sbin/init"        13 seconds ago      Up 10 seconds                           elasticsearch
cc1dc506908b        ansiblecira_jenkins_master   "/sbin/init"        13 seconds ago      Up 10 seconds                           jenkins_master
```

Now we'll fire off the playbook.

```
ansible-playbook site.yml -i hosts/containers -e use_openstack_deploy=false -e deploy_type='docker' -c docker
```

Alright, now you should be looking good, you'll see that there's some info about where Jenkins & Kibana UIs are located at the bottom, I went and pasted my snips here below:

```
TASK [Where is Kibana located?] ************************************************
ok: [kibana] => {
    "msg": "Kibana can be reached at http://172.20.0.4:5601/"
}

[... snip ...]

TASK [Where is Jenkins Master located?] ****************************************
ok: [jenkins_master] => {
    "msg": "Jenkins Master can be reached at http://172.20.0.2:8080/"
}

```

## Let's connect to the web UIs

If you're like me, this is running on a remote machine, and it's talking to a new bridge, and you don't have access to it over the network, so you'll have to tunnel in to reach them.

You'll find the IPs to use in the out put, and I tunnel like so:

```
[doug@localhost laboratoryb]$ ssh -L 5601:172.20.0.4:5601 stack@192.168.1.201
```

And point my browser on my local machine @ `http://localhost:5601`

...Although I don't have anything logged to ES, so it's complaining that there's nothing to find, but, I can get there!

And I can get to Jenkins similarly

```
[doug@localhost laboratoryb]$ ssh -L 8080:172.20.0.2:8080 stack@192.168.1.201
```

And that my friends is CIRA up and running! Another time we'll look about how to load it with jobs and how to create jobs to fit a need for testing an openstack reference architecture.