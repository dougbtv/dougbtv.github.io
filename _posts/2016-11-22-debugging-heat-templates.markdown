---
author: dougbtv
comments: true
date: 2016-11-22 13:34:00-05:00
layout: post
slug: debugging-heat-templates
title: TripleO Overcloud deploy failed? Much sadness. Debugging your isolated network configuration
category: nfvpe
---

In setting up my home lab, [Dr. Octagon](https://github.com/dougbtv/droctagon-ansible) I've been really stubbing my toe here and there getting the heat templates for network isolation just right. I still haven't come exactly to the conclusions that I need to come to, but in the process I've gained a better understanding of the process for debugging what's wrong with the templates in the process.

Some things have just been kicking me square in the keister and causing me [much sadness](https://cdn.meme.am/instances/750x/45457269.jpg). 

I've also collected a fair number of links & resources for helping you out in the meanwhile, too. Which I thought I'd share.

## Network Isolation Resources

* [TripleO Network Isolation document](http://docs.openstack.org/developer/tripleo-docs/advanced_deployment/network_isolation.html#using-the-native-vlan-for-floating-ips)
* [Red Hat docs on isolating networks](https://access.redhat.com/documentation/en/red-hat-openstack-platform/8/paged/director-installation-and-usage/62-isolating-networks)
* [THE TripleO Heat templates on GitHub](https://github.com/openstack/tripleo-heat-templates)

## Troubleshooting

* [Big props to Steve Hardy on debugging heat templates](http://hardysteven.blogspot.com/2015/04/debugging-tripleo-heat-templates.html) about debugging heat.
  * This, by far and wide, is the ultimate resource to use in this context.
* [TripleO "Troubleshooting a failed overcloud deploy"](http://docs.openstack.org/developer/tripleo-docs/troubleshooting/troubleshooting-overcloud.html)
  * This comes up almost every time you google something about your failed overcloud, huh? Does for me, too.
* [OpenStack Heat Troubleshooting guide](https://wiki.openstack.org/wiki/Heat/TroubleShooting)
* [fatmin's intro to troubleshooting heat](https://fatmin.com/2016/03/01/openstack-introduction-to-troubleshooting-heat/)
* [If you have O'Reilly Safari, OpenStack essentials has a chapter](https://www.safaribooksonline.com/library/view/openstack-essentials/9781783987085/ch13s12.html)
* [The RHOSP 7 heat troubleshooting guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/7/html/Director_Installation_and_Usage/sect-Troubleshooting_Overcloud_Creation.html)


### Some other references

Through the process I also referenced [cybertron's blog](http://blog.nemebean.com/content/network-isolation-tripleo-home-networks) and his [tripleo-network-templates](https://github.com/cybertron/tripleo-network-templates). He admits that they're out of date, but, was worthwhile to see another example (even if it didn't work off the shelf for me).

