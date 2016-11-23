---
author: dougbtv
comments: true
date: 2016-11-23 14:36:00-05:00
layout: post
slug: ironic-root-device
title: Specifying the root device with Ironic
---

So I was wondering why I got the fabled `No valid host was found. There are not enough hosts available` when adding a compute node to my cluster. And I spun my wheels for a while, and I saw with a `ironic node-show ${uuid}` that well... There was only 13 gig of local disk on the node. Wat!?!1! Yeah. That's not right, there's a 120 gig SSD in there. Is the disk bad? Let me boot off the USB stick on the box and try to use the HDD. Wait, I can't boot off that USB thumb drive? Yikes. Ohhhh, during deploy the USB stick got wiped! Henceforth, why you should specify a root device if you've got multiple storage devices.

So, I used the [setting the root device for deployment docs](http://docs.openstack.org/developer/tripleo-docs/advanced_deployment/root_device.html) to go ahead and specify what the root device is. I also found that the [TripleO docs on baremetal overclouds](http://docs.openstack.org/developer/tripleo-docs/advanced_deployment/baremetal_overcloud.html#enrolling-nodes) has a reference to this in the "Enrolling nodes" section.

Well first question I asked, what disk is it. There's a recipe for that, and you can see my `/dev/sda` is, well... Dangnabbit, the USB stick.

```
[stack@undercloud ~]$ openstack baremetal introspection data save 59a6fe44-899c-4907-ad8d-b104754a9a6a | jq '.inventory.disks'
[
  {
    "rotational": true,
    "vendor": "Kingston",
    "name": "/dev/sda",
    "wwn_vendor_extension": null,
    "wwn_with_extension": null,
    "model": "DT 101 G2",
    "wwn": null,
    "serial": "001372995FC8EC3025BF0031",
    "size": 15606349824
  },
  {
    "rotational": false,
    "vendor": "ATA",
    "name": "/dev/sdb",
    "wwn_vendor_extension": null,
    "wwn_with_extension": "0x5f8db4c391600cf2",
    "model": "PNY CS1311 120GB",
    "wwn": "0x5f8db4c391600cf2",
    "serial": "PNY39162193120100CF2",
    "size": 120034123776
  }
]
```

I thought we may need to go and specify some property in our instackenv. So let's do that now, and we'll try to import a single node to ironic. (Which is totally an experiment for me, I've only done my entire inventory at once, but, I don't wanna go too many steps backwards in my deployment right now). And just for the record -- the experiment failed. I couldn't get that to import properly with `openstack baremetal import single_node.json`. So I've selected a different route. Which we'll see below. However, I'm putting this into my playbook for [dr. octagon](https://github.com/dougbtv/droctagon-ansible) for now as an experiment, we'll see next time I import the entire instackenv.

I'm going to choose the serial number as the parameter I will use. And I had a simple `single_node.json` file that looked like:

```
{
  "nodes": [{
    "pm_type": "pxe_ipmitool",
    "mac": [
      "00:25:90:49:55:A6"
    ],
    "cpu": "2",
    "memory": "48130",
    "disk": "120",
    "root_device": {
      "serial": "PNY39162193120100CF2"
    },
    "arch": "x86_64",
    "pm_user": "stack",
    "pm_password": "redhatz",
    "capabilities": "profile:compute,boot_option:local",
    "pm_addr": "192.168.1.17"
  }]
}
```

I'm going to update the node manually for now.

```
[stack@undercloud ~]$ ironic node-update 59a6fe44-899c-4907-ad8d-b104754a9a6a add properties/root_device='{"serial": "PNY39162193120100CF2"}'
```

You can see that if you do a `ironic node-show ${uuid}` now, too.

I also re-ran introspection, with a process I'd' summarize as:

```
[stack@undercloud ~]$ ironic node-set-provision-state 59a6fe44-899c-4907-ad8d-b104754a9a6a manage
[stack@undercloud ~]$ openstack baremetal introspection start 59a6fe44-899c-4907-ad8d-b104754a9a6a
[stack@undercloud ~]$ ironic node-set-provision-state 59a6fe44-899c-4907-ad8d-b104754a9a6a provide
```

In the future what I'm going to do, to make for a sure thing `configure boot` with: 

```
openstack help baremetal configure boot --root-device-minimum-size 40
```

That should do it. Now.... To flash another live O/S on that thumb drive ;)
