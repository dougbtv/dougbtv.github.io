---
author: dougbtv
comments: true
date: 2016-11-09 13:39:00-05:00
layout: post
slug: openshift-ansible-byo-inventory
title: openshift-ansible - Bring your own inventory.
category: nfvpe
---

I'm working on getting an OpenShift cluster up and running, and in the while of trying to do it properly with [openshift-ansible](https://github.com/openshift/openshift-ansible)'s openstack method (which uses heat templates, as opposed to us building it all by hand here), I was trouble shooting an issue (my clocks were.... quite wrong, causing me much grief with creating proper certs for HTTPS necessary for etcd)... And I wound up going and doing the "create your own inventory" method.

I don't recommend you *start* with this method, but, I'm putting together my documentation for how I did it here, as it does prove useful in it's own way.

I recommend you stay tuned for the next article about how to do it in a way that I believe will be a fair bit more graceful (even if it does have it's own considerations to work with). There's a few things here that we're babying that don't necesarily need to be so hard.

This article assumes you have a triple-o instance up, which you can use my ["easy mode"](http://dougbtv.com/nfvpe/2016/10/31/openshift-on-openstack-easy-mode/) article to get you started on. And the next article will give you some handy hints about how to build on that original triple-o instance to get it groomed a bit for what we're doing here.

## Security groups

One thing we lose from bringing our own inventory is that we need to create security groups.  Let's go ahead and create these.

```
source ~/overcloudrc

# etcd group
nova secgroup-create openshift-etcd "etcd group for openshift"
nova secgroup-add-rule openshift-etcd tcp 22 22 0.0.0.0/0
nova secgroup-add-rule openshift-etcd tcp 2379 2379 0.0.0.0/0
nova secgroup-add-rule openshift-etcd tcp 2380 2380 0.0.0.0/0
nova secgroup-add-rule openshift-etcd icmp -1 -1 0.0.0.0/0

# infra group
nova secgroup-create openshift-infra "infra group for openshift"
nova secgroup-add-rule openshift-infra tcp 22 22 0.0.0.0/0
nova secgroup-add-rule openshift-infra tcp 80 80 0.0.0.0/0
nova secgroup-add-rule openshift-infra tcp 443 443 0.0.0.0/0
nova secgroup-add-rule openshift-infra icmp -1 -1 0.0.0.0/0

# master group
nova secgroup-create openshift-master "master group for openshift"
nova secgroup-add-rule openshift-master tcp 22 22 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 53 53 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 2224 2224 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 4001 4001 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 8053 8053 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 8443 8443 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 8444 8444 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 9090 9090 0.0.0.0/0
nova secgroup-add-rule openshift-master tcp 24224 24224 0.0.0.0/0
nova secgroup-add-rule openshift-master udp 53 53 0.0.0.0/0
nova secgroup-add-rule openshift-master udp 5404 5404 0.0.0.0/0
nova secgroup-add-rule openshift-master udp 5405 5405 0.0.0.0/0
nova secgroup-add-rule openshift-master udp 8053 8053 0.0.0.0/0
nova secgroup-add-rule openshift-master udp 24224 24224 0.0.0.0/0
nova secgroup-add-rule openshift-master icmp -1 -1 0.0.0.0/0

# compute group
nova secgroup-create openshift-compute "compute group for openshift"
nova secgroup-add-rule openshift-compute tcp 22 22 0.0.0.0/0
nova secgroup-add-rule openshift-compute tcp 10250 10250 0.0.0.0/0
nova secgroup-add-rule openshift-compute tcp 10255 10255 0.0.0.0/0
nova secgroup-add-rule openshift-compute tcp 30000 32767 0.0.0.0/0
nova secgroup-add-rule openshift-compute udp 4789 4789 0.0.0.0/0
nova secgroup-add-rule openshift-compute udp 10255 10255 0.0.0.0/0
nova secgroup-add-rule openshift-compute icmp -1 -1 0.0.0.0/0

```

## Spin up some nova instances

Alright, let's go ahead and spin up 4 nova instances that we'll use to run OpenShift Origin on. Here we define some lists of the nodes, and what security groups they belong to, as well as their flavors.

You'll note here that we're using our "atomic_key" which we created in the "easy mode" article, it's also in the inventory in the next steps.

```
#!/bin/bash

# As you must source it.
source ~/overcloudrc

# Find our internal network.
internal_net=$(neutron net-list | awk ' /int/ {print $2;}')

# Define what nodes we have, and what security groups they belong to.
nodelist="openshift-compute-0 openshift-compute-1 openshift-master-0 openshift-infra-0"
declare -A security_group
security_group[openshift-compute-0]=openshift-compute
security_group[openshift-compute-1]=openshift-compute
security_group[openshift-master-0]=openshift-master
security_group[openshift-infra-0]=openshift-infra

# Define what flavor we'll use for each node.
declare -A flavor
flavor[openshift-compute-0]=m1.medium
flavor[openshift-compute-1]=m1.medium
flavor[openshift-master-0]=m1.small
flavor[openshift-infra-0]=m1.small

# Cycle those defined nodes
for node in $nodelist; do
  # Boot an instance.
  nova boot --flavor ${flavor[$node]} --key-name atomic_key --nic net-id=$internal_net --image centos-timezone $node > /dev/null
  # Allocate a floating IP
  new_ip=$(neutron floatingip-create ext-net | grep floating_ip_address | awk '{print $4}')
  # Assign the floating IP
  nova add-floating-ip $node $new_ip
  # Add it to a security group.
  nova add-secgroup $node ${security_group[$node]}
  # Echo out an ansible host list for inventory.
  echo "$node ansible_host=$new_ip"
done
```

The output of this list will result in a header for an ansible inventory file (assuming it works!) mine wound up looking like:

```
openshift-compute-0 ansible_host=192.168.24.105
openshift-compute-1 ansible_host=192.168.24.106
openshift-master-0 ansible_host=192.168.24.107
openshift-infra-0 ansible_host=192.168.24.108
```

## Create yourself an inventory file.

Here's the ansible inventory file, I'll put mine in `~/inventory`

```
openshift-compute-0 ansible_host=192.168.24.105
openshift-compute-1 ansible_host=192.168.24.106
openshift-master-0 ansible_host=192.168.24.107
openshift-infra-0 ansible_host=192.168.24.108

# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_ssh_private_key_file=/home/stack/atomic_key.pem
ansible_ssh_user=centos
ansible_become=true
become_user=root

# deployment type valid values are origin, online and enterprise
deployment_type=origin
# htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/openshift/htpasswd'}]
# default subdomain to use for exposed routes
osm_default_subdomain=atomicapps.miminar.local
# host group for masters

[masters]
openshift-master-0

[nodes]
openshift-infra-0 openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
openshift-compute-0 openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
openshift-compute-1 openshift_node_labels="{'region': 'primary', 'zone': 'west'}"
```

## Prep to run openshift-ansible (from the undercloud)

Let's install the rest of what we need. (note: The `python-*` clients were already installed)

```
ssh -F /home/doug/.quickstart/ssh.config.ansible stack@undercloud
. ~/overcloudrc

# Here's our package deps (need Ansible >= 2.1.0, 2.2 preferred, and Jinja >= 2.7)
sudo yum install -y ansible pyOpenSSL python-cryptography python-novaclient python-neutronclient python-heatclient

# Apparently we needed this to run the cluster creator (wasn't in the docs)
sudo pip install shade

# And get the openshift-ansible repo.
git clone https://github.com/openshift/openshift-ansible.git
cd openshift-ansible/
```

## Let's double check some stuff

What the heck times do we have across boxen?

```
for i in $(seq 109 112); do ssh -k ~/atomic_key centos@192.168.24.$i "hostname && date"; done
```

## Run the BYO playbook

Beyond this point... Your mileage may vary! Stay tuned for the next installment, where we'll dig into this in some better detail.

```
[stack@undercloud openshift-ansible]$ ansible-playbook -i ~/inventory ./playbooks/byo/config.yml
```
