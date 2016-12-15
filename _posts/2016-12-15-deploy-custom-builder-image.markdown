---
author: dougbtv
comments: true
date: 2016-12-15 17:00:00-05:00
layout: post
slug: deploy-custom-builder-image
title: Deploy a custom builder image on OpenShift
category: nfvpe
---

In the last article on [creating custom s2i builder images](http://dougbtv.com/nfvpe/2016/12/09/openshift-s2i-custom-builder/) we created the (intentionally ridiculous) `pickle-http` sample, and today we're going to go ahead and deploy it under openshift. It's the easy part, when it comes down to it. It's rather fast, and cockpit (the web GUI) provides some nice clean information about the builds, including logs and links to webhooks to trigger builds.

## Push custom builder image to registry

First I went ahead and pushed my local image to a public image in this case (you can push it to your local registry if you want, or you can feel free to use the public image named `bowline/pickle-http`). I tagged the image, pushed it -- oh yeah, and I logged into dockerhub (not shown).

```
[openshift@test-cluster-master-0 stackanetes]$ docker tag pickle-http bowline/pickle-http
[openshift@test-cluster-master-0 stackanetes]$ docker push bowline/pickle-http
```

## Create a new project and deploy new app!

Next I created a play project to work under in openshift, I also added this role to the admin user, so that I can see the project on cockpit.

```
[openshift@test-cluster-master-0 stackanetes]$ oc new-project pickler
[openshift@test-cluster-master-0 stackanetes]$ oc policy add-role-to-user admin admin -n pickler
```

Then we create a new app using our custom builder image. This is... as easy as it gets.

```
[openshift@test-cluster-master-0 stackanetes]$ oc new-app bowline/pickle-http~https://github.com/octocat/Spoon-Knife.git
```

Basically it's just in the format `oc new-app ${your_custom_builder_image_name}~{your_git_url}`.

## Inspect the app's status and configure a readiness probe

It should be up at this point (after a short wait to pull the image). Great! It's fast. Really fast, how great. Granted, we have the simplest use case -- "just clone the code into my container". So in this particular case if you don't have the image pulled yet, that's going to be the longest wait.

Let's look at its status.

```
[openshift@test-cluster-master-0 stackanetes]$ oc status
In project pickler on server https://192.168.17.4:8443

svc/spoon-knife - 172.30.236.145:8080
  dc/spoon-knife deploys istag/spoon-knife:latest <-
    bc/spoon-knife source builds https://github.com/octocat/Spoon-Knife.git on istag/pickle-http:latest 
    deployment #1 deployed 6 minutes ago - 1 pod

1 warning identified, use 'oc status -v' to see details.
```

We have a warning, and it's because we don't have a "[readiness probe](https://docs.openshift.com/enterprise/3.1/dev_guide/application_health.html#container-health-checks-using-probes)". A "probe" is a k8s action that can take diagnostic actions periodically. Let's go ahead and add ours to be complete. 

Pick up on some help on the topic with:

```
[openshift@test-cluster-master-0 stackanetes]$ oc help set probe
```

```
oc set probe dc/spoon-knife --readiness --get-url=http://:8080/
```

In this case we'll just look at the index on port 8080. You can run `oc status` again and see that we're clear.


## Look at the details of the build on cockpit

Now that we have a custom build going for us, there's a lot more on the UI that's going to be interesting to us. Firstly navigate to `Builds -> Builds`. From there choose "spoon-knife". 

There's a few things here that are notable:

* Summary -> Logs: check out what happened in the s2i custom building process (in this case, just a git clone)
* Configuration: Has links to triggers to automatically trigger a new build (e.g. in a git webhook), details on the git source repository

That's that, now you can both create your own custom builder image, and go forward with deploying pods crafted from just source (no dockerfile!) on openshift.
