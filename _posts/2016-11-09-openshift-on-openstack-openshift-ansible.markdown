---
author: dougbtv
comments: true
date: 2016-11-09 17:16:10-05:00
layout: post
slug: openshift-on-openstack-openshift-ansible
title: OpenShift-on-OpenStack - Creating a cluster with openshift-ansible
category: nfvpe
---

We're going to try to use the [openshift-ansible](https://github.com/openshift/openshift-ansible) to spin up an OpenShift cluster. In the "easy mode" article about OpenShift, we created just a single play node, and also a whopping single point of failure. So let's build up on that. We'll take this to the point where we can see the pods that are running, and see where the dashboard is, and in a further article explore how to set up the networking and at least make it a little more real-world-useable.

Before you begin, I recommend you check out the process to [build a custom CentOS cloud image](http://dougbtv.com/nfvpe/2016/11/03/custom-centos-cloud-image/) as it's going to be required here.

---
# Openstack setup

Using the ["easy mode" article](http://dougbtv.com/nfvpe/2016/10/31/openshift-on-openstack-easy-mode/) as a starting point, get yourself an overcloud up with oooq.

However, with a little different twist. That setup just doesn't have enough horsepower, errr, ram power mostly that is. So I'm gonna deploy more compute nodes. We're going to use a custom `general_config` file in `./tripleo-quickstart/config/general_config/doug.yml`. I went ahead and copied `minimal.yml` and added the parameters I need. While I'm still learning about oooq, I do believe that these are overrides for variables that are used throughout the Ansible plays.

Here's the variables that I set differently (and here's my [entire doug.yml file if you please](https://gist.github.com/dougbtv/6d1e63acd9e037899fc4d0a684669e60)):

```
# Top level vars, right up near the top.
control_memory: 8192
compute_memory: 12288
undercloud_memory: 16384

compute_vcpu: 4
compute_disk: 100

# Added to "overcloud_nodes"
  - name: compute_1
    flavor: compute
  - name: compute_2
    flavor: compute

# I just added the compute-scale here
extra_args: >-
  --compute-scale 3
  --neutron-network-type vxlan
  --neutron-tunnel-types vxlan
  --ntp-server pool.ntp.org
```

I ran into a number of issues from having the undercloud (especially) being light on RAM. I over provisioned for what my test system has, and so far my experience has been that it's better to overprovision in the smaller environment I'm using now (single host, 8 cores, 32 gig RAM).

When you've got that new yaml file all setup, go ahead and run oooq as such:

```
./quickstart.sh --config ./config/general_config/doug.yml eendracht
```

(And sometimes I add a `--extra-vars "force_cached_images=true"` to skip re-downloading.)

Don't forget your overcloud images & introspection.

```
ssh -F /home/doug/.quickstart/ssh.config.ansible undercloud
source stackrc
openstack overcloud image upload
openstack baremetal import instackenv.json
openstack baremetal configure boot
openstack baremetal introspection bulk start
```

And also chuck in the subnet DNS.

```
subnet_id=$(neutron subnet-list | grep -i "start" | awk '{print $2}')
neutron subnet-update $subnet_id --dns-nameserver 8.8.8.8
```

When we get to the deploy we're going to do the deploy in multiple parts since we're working with a bigger cluster. We're just going to set the compute scale to 1. Side note: I actually tend to do this in a screen now, and also `tee` the output to a log. That way if I lose network connection, it still keeps running and I'm not left wondering, "where'd that go, and did it finish?"


```
openstack overcloud deploy --templates --compute-scale 1 --control-scale 1
```

And then increase the scale.

```
openstack overcloud deploy --templates --compute-scale 3  --control-scale 1
```

Ok, we've got an overcloud! Mine resulted in this URI.

```
Overcloud Endpoint: http://192.168.24.6:5000/v2.0
```

So you can get to the OpenStack web GUI with tunneling a la...

```
ssh -F ~/.quickstart/ssh.config.ansible stack@undercloud -L 8080:192.168.24.6:80
```

And we'll create our network again according to the "easy mode" article (I noticed that in `#oooq` they mentioned the default netmask changed, so I put my updated steps here)

```
neutron net-create ext-net --router:external --provider:physical_network datacentre  --provider:network_type flat
neutron subnet-create ext-net --name ext-subnet --allocation-pool start=192.168.24.100,end=192.168.24.120 --disable-dhcp --gateway 192.168.24.1  192.168.24.0/24
neutron router-create router1
neutron router-gateway-set router1 ext-net
neutron net-create int
neutron subnet-create int 30.0.0.0/24 --dns_nameservers list=true 8.8.8.8
neutron router-interface-add router1 $ABOVE_ID
```

Or, even better, I made [a gist of a bash script I use to create my network](https://gist.github.com/dougbtv/046b8a706d03d2737b4048c45d92a70b), for now. Make sure to change the constants at the top of the script to match what you prefer.

If you'd like, spin up an instance and give it a test.

How about a quick checklist of what your ooo deployment looks like before we move on?

* oooq undercloud
* overcloud image upload, introspection
* oooq overcloud deploy
* setup the networking as above
* add some default security group rules
* upload the custom image to glance
* make sure you have a nova key made.
* spin up an instance as a test.

All of these steps can found between here and the "easy mode" article.

---

## Prep to run openshift-ansible (from the undercloud)

Let's install the rest of what we need. (note: The `python-*` clients were already installed)

```
ssh -F /home/doug/.quickstart/ssh.config.ansible stack@undercloud
. ~/overcloudrc

# Here's our package deps (need Ansible >= 2.1.0, 2.2 preferred, and Jinja >= 2.7)
sudo yum install -y ansible pyOpenSSL python-cryptography python-novaclient python-neutronclient python-heatclient

# Apparently we needed this to run the cluster creator (wasn't in the docs, but is now after a PR)
sudo pip install shade

# And get the openshift-ansible repo.
git clone https://github.com/openshift/openshift-ansible.git
cd openshift-ansible/
```

And let's also modify the play that creates the heat stack. The timeout is too low.

```
sed -i 's/timeout 3/timeout 10/' playbooks/openstack/openshift-cluster/launch.yml
```

----
## OpenShift cluster creator method and pitfalls.

tl;dr -- If you're not concerned with the gotchyas, just move to the next section with what to do next. But come back here if you get stuck somewhere.

First off -- we're going to be performing these actions from our undercloud machine, as we've done everything else in this article so far. It has the basics that we need, and it can easily access the overcloud. The gist is that we're going to SSH to that machine, setup the requirements, then source `. ~/overcloudrc` and kick off the playbooks.

The method we're using here is to use the openstack features of the `./bin/cluster` application in openshift-ansible -- here's the official [README_openstack.md](https://github.com/openshift/openshift-ansible/blob/master/README_openstack.md). 

All of these Markdown files give a standard *"This feature is community supported and has not been tested by Red Hat"* warning. Because they'd prefer that you use the ["out-of-the-box" methods of installing OpenShift](https://install.openshift.com/), but, we'd like to use OpenShift Origin (the upstream project) so we're going a little out of the box here. The "officially supported" (if you can call it that) method described there for Origin is to use the `oc` (OpenShift Command) application, which spins up an "all-in-one" instance of OpenShift Origin, and we want a cluster! 

If you're looking for another reference on how bring up a cluster with OpenShift Origin, may I suggest you checkout the the [cluster instructions from the advanced installation docs](https://docs.openshift.org/latest/install_config/install/advanced_install.html#install-config-install-advanced-install) (which... actually uses ansible, itself.) Alternatively, you can aslo "bring your own host deployment" according to the openshift-ansible docs. 

Currently I've run into a few issues that I've opened issues and PR's for (err, PR's for most), which I have somewhat fixed herein. At some point, I may circle around and try to improve the playbooks and submit a PR so that this happens more smoothly. However, after working through most of the kinks (as described here), I feel it's less necessary.

**One, if the heat create succeeds, but, further playbook errors occur, rebuilding the heat stack is nearly impossible with the playbook.**

So I've gone ahead and created a script that helps you manually hoe out the networking created by ansible openshift. It just forcefully tears it out.

The teardown script is available [in this gist](https://gist.github.com/dougbtv/9c79dde81b1e35208c85bb2924dc1c96). Remember to change the variables up at the top, the most important being the cluster name you use in the following commands to spin up openshift-ansible with the cluster creator.

Remember to follow up with a `heat stack-list` and then `heat stack-delete $your_cluster` when you're done, as it doesn't do that step.

**Two, the timeout can be too low for the heat stack creation**.

I've included a sed command herein to fix it for you, but, I've also opened a [PR on github to add a --timeout option](https://github.com/openshift/openshift-ansible/issues/2733).

**Last and (sorta) least, we're missing a python module, shade**

The docs for [openshift-ansible with openstack](https://github.com/openshift/openshift-ansible/blob/master/README_openstack.md) didn't originally tell you you're going to need to install a Python module, so I include it in the issues here and in my instruction, but they have merged my [documentation fix in this PR](https://github.com/openshift/openshift-ansible/pull/2732).

----
## Run the cluster creator

Let's move on to brewing up our install command. You'll note here that we're specifying the `centos-custom` image, which we created in [this article](http://dougbtv.com/2016/11/03/custom-centos-cloud-image/).

```
bin/cluster create \
  -o image_name=centos-custom \
  -o external_net=ext-net \
  -o floating_ip_pool=ext-net \
  -o net_cidr=40.0.0.0/24 \
  openstack test_cluster
```

If you'd like more nodes add the `-n $number_of_nodes` to your cluster create command.

Here's my list... (I reformatted mine, fwiw.)

```
role      -  private IP  -    public IP               
master    - 192.168.137.4 - 192.168.24.104
compute 0 - 192.168.137.6 - 192.168.24.106
compute 1 - 192.168.137.5 - 192.168.24.105
infra     - 192.168.137.3 - 192.168.24.103
```

----
## Accessing OpenShift

Ok, let's ssh into the master, since we didn't specify any specific SSH keys above, if we are using the stack user in our undercloud, it's going to have the right key. Note that we ssh as the `openshift` user.

```
[stack@undercloud ~]$ ssh openshift@192.168.24.104
```

Let's take a look at the health of the cluster, using the `oc` OpenShift command...

```
oc get nodes
```

If you're like me, you're unlucky, your nodes are going to say "NotReady", like this:

```
[openshift@test-cluster-master-0 ~]$ oc get nodes
NAME            STATUS                        AGE
192.168.137.3   NotReady                      1h
192.168.137.4   NotReady,SchedulingDisabled   1h
192.168.137.5   NotReady                      1h
192.168.137.6   NotReady                      1h
```

So I went to each node and restarted the `origin-node` service.

```
[openshift@test-cluster-node-compute-1 ~]$ sudo systemctl restart origin-node
```

Or, do them all in one sweep... 

```
for i in $(seq 103 106); do ssh openshift@192.168.24.$i "sudo systemctl restart origin-node"; done
```


After doing that for each of those nodes they look like:

```
[openshift@test-cluster-master-0 ~]$ oc get nodes
NAME            STATUS                     AGE
192.168.137.3   Ready                      1h
192.168.137.4   Ready,SchedulingDisabled   1h
192.168.137.5   Ready                      1h
192.168.137.6   Ready                      1h
```

Since `192.168.137.4` is my master, I think I'm OK with "scheduling disabled" for now, I'm guessing I don't want to schedule containers there for the moment.

----
## OpenShift Projects

Ok, still on the master, let's look at the projects available, you can do this with `oc projects` command

```
[openshift@test-cluster-master-0 ~]$ oc projects
You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-system
    logging
    management-infra
    openshift
    openshift-infra

Using project "default" on server "https://192.168.92.6:8443".
```

Let's go ahead and ensure we're on the "default" project by issuing a command to change the active project (you may already be on it, by, uh, default)

```
[openshift@test-cluster-master-0 ~]$ oc project default
Now using project "default" on server "https://192.168.92.6:8443".
```

Now let's look at the pods that are there. 

```
[openshift@test-cluster-master-0 ~]$ oc get pods
NAME                       READY     STATUS    RESTARTS   AGE
docker-registry-5-95oai    1/1       Running   0          3m
registry-console-1-22xf5   1/1       Running   0          3m
router-1-p7do8             1/1       Running   0          3m
```

More detail [on what pods are](http://kubernetes.io/docs/user-guide/pods/) another time. For now, let's inspect one.

If you're zipping through this document and having good luck, you might see the `READY` column reads `0/1` which means it's not ready and status may not be "Running" this is OK too. It takes some time (especially while the Docker images download). Still go ahead and do a `oc describe pod $name` and check it out, it's also interesting.

Let's look at that.... "registry" one

```
[openshift@test-cluster-master-0 ~]$ oc describe pod registry-console-1-22xf5 | grep -iP "^IP|^\s+Port"
IP:     10.129.0.2
    Port:   9090/TCP
```

Oh, interesting.... It's TCP running on port 9090 @ 10.129.0.2

Let's curl that....

```
curl -k https://10.129.0.2:9090
```

That my friend, is cockpit -- your openshift dashboard. Kubenernetes is greek for pilot -- so, how fitting, the cockpit.

Wouldn't it be great if you could easily hook up your browser to that? It sure would be. In the next article we'll focus on how to make this networking setup more useable.


