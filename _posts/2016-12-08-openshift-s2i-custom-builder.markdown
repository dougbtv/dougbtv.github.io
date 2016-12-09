---
author: dougbtv
comments: true
date: 2016-12-09 15:59:10-05:00
layout: post
slug: openshift-s2i-custom-builder
title: Using OpenShift's s2i custom builder
category: nfvpe
---

Let's use OpenShift's s2i [custom building functionality](https://docs.openshift.com/enterprise/3.0/creating_images/custom.html) to make a [custom image build](https://blog.openshift.com/create-s2i-builder-image/). This walk-through assumes that you have an openshift instance up and running. I have a couple tutorials available on this blog, or you can [just run an all-in-one server](https://docs.openshift.org/latest/getting_started/administrators.html).

A little background. I'm exploring a few different build pipelines for Docker images, in a couple different cases (one of which being [CIRA](https://github.com/redhat-nfvpe/ansible-cira)). Naturally my own [Bowline](https://github.com/dougbtv/bowline) comes to mind, and I think it still does fit a particularly good need for both build visibility / build logs, and also for publishing images. However, I'd like to explore the options with doing it all within OpenShift.

Our goal here is going to be to make an image for a custom HTTP server that shows a raster graphic depicting a pickle, and then we can make a "custom application" which is a git repo that gets cloned into here and we can serve up *more* pickle images in this case. So our custom dockerfile has an `index.html` and a single pickle graphic, and then when a custom build is triggered we clone in some pickles that can be viewed (Why all the pickles? Mostly just because it's custom, is really all, and at least mildly more entertaining than just saying "hello world!"). For now we're just going to build the image, and run it manually. In another installment we'll feed this into OpenShift Origin proper and use a builder image there and deploy a pod.

I have an openshift cluster up, and I'm going to ssh into my master and perform these operations there.

## Installing s2i

So you ssh'd to your master, and you tried running `s2i` for fun, and "command not found", so you do `which s2i` and it's not there. Sigh. It's a stand-alone tool, so you'll have to install.

Go ahead and browse to the [latest version](https://github.com/openshift/source-to-image/releases/latest) and let's download the tar, extract it, and move the bins into place.

```
[openshift@test-cluster-master-0 ~]$ curl -L -O https://github.com/openshift/source-to-image/releases/download/v1.1.3/source-to-image-v1.1.3-ddb10f1-linux-386.tar.gz
[openshift@test-cluster-master-0 ~]$ tar -xzvf source-to-image-v1.1.3-ddb10f1-linux-386.tar.gz
[openshift@test-cluster-master-0 ~]$ sudo mv {s2i,sti} /usr/bin/
[openshift@test-cluster-master-0 ~]$ s2i version
```

## Setup the Dockerfile

Run `s2i` to create a `Dockerfile` and s2i templates for you. You'll note that the first argument after create "pickle-http" is the name of the image, and second and last argument is the name of the directory it creates

```
[openshift@test-cluster-master-0 ~]$ s2i create pickle-http s2i-pickle-http
[openshift@test-cluster-master-0 ~]$ cd s2i-pickle-http/
[openshift@test-cluster-master-0 s2i-pickle-http]$ ls -l
total 8
-rw-------. 1 openshift openshift 1257 Dec  9 09:53 Dockerfile
-rw-------. 1 openshift openshift  175 Dec  9 09:53 Makefile
drwx------. 3 openshift openshift   48 Dec  9 09:53 test
[openshift@test-cluster-master-0 s2i-pickle-http]$ find
```

You'll note in the `find` that the `s2i` command has bootstrapped a bunch of assets for us.

Important.... Now let's download our pickle photograph. (Feel free to download your own pickle.)

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ curl -o ./pickle.jpg http://i.imgur.com/m8R5SJX.jpg
```

And let's create a `index.html` file

```
[openshift@test-cluster-master-0 ~]$ cat << EOF > index.html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
  <head>
    <meta http-equiv="Content-Type" content="text/html;charset=ISO-8859-1" />
    <title>Pickle Raster Graphic</title>
  </head>
  <body>
    <p>
      This is a <a href="/images/">collection of pickles</a>.
    </p>
    <p>
      <img src="/pickle.jpg" alt="that's a pickle." />
    </p>
  </body>
</html>
EOF
```

Let's go ahead and edit the `Dockerfile`. Here's what my `Dockerfile` looks like now.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ cat Dockerfile 
# pickle-http
FROM centos:centos7
MAINTAINER @dougbtv
ENV BUILDER_VERSION 1.0
LABEL io.k8s.description="It shows a freakin' pickle, dude." \
      io.k8s.display-name="pickle 0.2.4" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="pickle,preservation,demo,food,cucumber"

# Install apache and add our content
RUN yum install -y httpd && yum clean all -y
ADD index.html /var/www/html/
ADD pickle.jpg /var/www/html/
RUN mkdir -p /var/www/html/images/

# Configure apache to use port 8080 (this simplifies some OSP stuff for us)
RUN sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

# TODO (optional): Copy the builder files into /opt/app-root
# COPY ./<builder_folder>/ /opt/app-root/

# Add the s2i scripts.
LABEL io.openshift.s2i.scripts-url=image:///usr/libexec/s2i
COPY ./.s2i/bin/ /usr/libexec/s2i

# Setup privileges for both s2i code insertion, and openshift arbitrary user
RUN mkdir -p /opt/app-root/src
ENV APP_DIRS /opt/app-root /var/www/ /run/httpd/ /etc/httpd/logs/ /var/log/httpd/
RUN chown -R 1001:1001 $APP_DIRS
RUN chgrp -R 0 $APP_DIRS
RUN chmod -R g+rwx $APP_DIRS

WORKDIR /opt/app-root/src

# This default user is created in the openshift/base-centos7 image
USER 1001

EXPOSE 8080

CMD /usr/sbin/httpd -D FOREGROUND
```

## Modifying the s2i scripts

Great, let's look at the `./.s2i/bin/assemble` file, go ahead and `cat` that if you'd like. This is responsible for building the application.

I just added a single line to mine to say to copy the cloned git repo into the `/var/www` document root for apache.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ tail -n 2 ./.s2i/bin/assemble
# TODO: Add build steps for your application, eg npm install, bundle install
cp -R /tmp/src/* /var/www/html/images
```

Now, time to modify the `run` s2i script. In this case we'll just basically be doing apache in the foreground. Here's what mine looks like now.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ cat ./.s2i/bin/run
#!/bin/bash -e
#
# S2I run script for the 'pickle-http' image.
# The run script executes the server that runs your application.
#
# For more information see the documentation:
# https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

exec /usr/sbin/httpd -D FOREGROUND
```

There's also a construct called "incremental builds" that we're not using, so we're going to remove that script.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ rm ./.s2i/bin/save-artifacts
```

There's also a usage script that you can decorate to make for better usage instructions. We're going to leave ours alone for now, but, you should update yours! Here's where it is.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ cat ./.s2i/bin/usage
```

## Build all the things!

First off, go ahead and build your `pickle-http` s2i image. 

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo docker build -t pickle-http .
```


Let's make a little placeholder, and put something that's not exactly a pickle image into the `test/test-app` dir.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ echo "not a pickle image" > test/test-app/pickle.txt
```

Now we can run s2i to import code into this image.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo s2i build test/test-app/ pickle-http sample-pickle
```

In likely reality you'll be cloning a git repo into this bad boy, here we'll use the [sample "spoon-knife" git repo to learn forking on github](https://github.com/octocat/Spoon-Knife) and it'll look more like...

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo s2i build https://github.com/octocat/Spoon-Knife.git pickle-http sample-pickle
```

Go ahead and finish this up using the git method there.

## Run the image to test it

Alright, so now we have a few images, if the above has been going well for you, you should have a set of docker images that looks something like:

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
sample-pickle       latest              ffaa100d8fa2        About a minute ago   246.6 MB
pickle-http         latest              8e03db77f8e6        7 minutes ago        246.6 MB
docker.io/centos    centos7             0584b3d2cf6d        5 weeks ago          196.5 MB
```

Let's go ahead and give that a run...

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo docker run -u 1234 -p 8080:8080 -d sample-pickle 
```

Wait... wait... Why are you using the `-u` parameter to run as user #1234? That my friend is to test to make sure this will actually run on OpenShift. Since OpenShift is going to pick an arbitrary user to run this image as, we're going to test it here with a faked out user id. I've accounted for this in the Dockerfile above.

If all is working well, you should have it in your `docker ps` and you can see it's logs

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
2f650727ef43        sample-pickle       "/usr/libexec/s2i/run"   15 seconds ago      Up 14 seconds       0.0.0.0:8080->8080/tcp   distracted_bassi
[openshift@test-cluster-master-0 s2i-pickle-http]$ sudo docker logs distracted_bassi
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.131.0.2. Set the 'ServerName' directive globally to suppress this message
```

Now let's go ahead and see that's it is actually serving our content. This command should show the index HTML that we baked into the base image.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ curl localhost:8080
```

We have dynamic content in the `/images` directory in the document root, so let's look at what's there.

```
[openshift@test-cluster-master-0 s2i-pickle-http]$ curl -s localhost:8080/images/ | grep -i fork.me
  Fork me? Fork you, @octocat!
```

You can see that it's the content from the git clone running in the container created from the `sample-pickle` docker image.

In another article, we'll go into how to add this builder image to your running openshift cluster so that you can deploy pods/containers using it.

## Editorial

The way this is made is rather rapid when it comes to inserting the code, granted -- we're just adding some content to a simple flat HTTP server. I think this might be just the ticket if you're deploying (say) 100 microservices all in the same programming language. Or 100 microservices across 5 programming languages. This could be very convenient.

I'll probably have more to say when it comes to deploying the builder image, and I think this is fairly handy. Where I still like Bowline is that it's rather visual, and gives good visibility of a build process. It also has solid logging to show you what's happening in your builds, and has a lot of opportunities for extensibility. They're really... two different kind of tools. 