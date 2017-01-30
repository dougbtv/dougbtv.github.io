---
author: dougbtv
comments: true
date: 2017-01-18 09:28:01-05:00
layout: post
slug: kpm-registry
title: Bootstrap a kpm registry to run a kpm registry
---

Yo dawg... I heard you like kpm-registries. So I bootstrapped a kpm-registry so you can deploy a kpm-registry from a kpm-registry.

So, I was deploying my kpm registry using a public, and beta kpm registry, and this happens right about the time I'm about to give a demo of spinning up stackanetes, and I need a kpm registry for that... But, the beta kpm registry (beta.kpm.sh) was down, argh/fiddlesticks!. So I went through and deploy a kpm registry so I can push a kpm registry package to run it. In the meanwhile I also opened a [kpm issue](https://github.com/coreos/kpm/issues/148), too.

Why the extra steps here, like... If you can run a kpm registry without a kpm registry, why would you do it? The thing is... Then I'm managing it myself (between a single docker container and a gunicorn web app), instead of having Kubernetes (k8s) manage it for me. And I want k8s to do the work. So I just bootstrap it, then I can deploy it as k8s pods.

This already assumes that you have kpm (the client) installed. If you don't have kpm installed, go ahead and [use my ansible galaxy role](https://galaxy.ansible.com/dougbtv/kpm-install/) to do so. Which will give you a clone of the kpm client in `/usr/src/kpm/`

Also make sure you have gunicorn (the "[green unicorn](http://gunicorn.org/)", a Python web server gateway interface) installed.

```
$ sudo yum install -y python-gunicorn
```

It requires etcd to be present, so get up etcd first.

```
$ docker run --name tempetcd -dt -p 2379:2379 -p 2380:2380 quay.io/coreos/etcd:v3.0.6 /usr/local/bin/etcd -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 -advertise-client-urls http://$127.0.0.1:2379,http://127.0.0.1:4001
```

Now you can run the registry API server with gunicorn, a la:

```
$ pwd
/usr/src
$ gunicorn kpm.api.wsgi:app -b :5555
```

And then you can push the kpm-registry packages, but only after you set the proper tag in the manifest, because there isn't a pushed image for this particular tag.

```
$ pwd
/usr/src/kpm/deploy/kpm-registry
$ sed -i 's/v0.21.2/v0.21.1/' manifest.jsonnet 
$ kpm push -H http://localhost:5555 -f
package: coreos/kpm-registry (0.21.2-4) pushed
```

Can we it deploy kpm-registry now? Not quite... We also have to push the `coreos/etcd` package to our bootstrapping registry. And I found the manifest for it in the `kubespray/kpm-packages` repo.

```
$ cd /usr/src/
$ git clone https://github.com/kubespray/kpm-packages.git
$ cd kpm-packages/
$ cd coreos/etcdv3
$ pwd
/usr/src/kpm-packages/coreos/etcdv3
$ kpm push -H http://localhost:5555 -f
$ kpm list -H http://localhost:5555
app                  version    downloads
-------------------  ---------  -----------
coreos/etcd          3.0.6-1    -
coreos/kpm-registry  0.21.2-4   -
```

Now you should be able to deploy a kpm registry from the bootstrapping registry via:

```
$ kpm deploy coreos/kpm-registry --namespace kpm -H http://localhost:5555
create coreos/kpm-registry 

 01 - coreos/etcd:
 --> kpm (namespace): created
 --> etcd-kpm-1 (deployment): created
 --> etcd-kpm-2 (deployment): created
 --> etcd-kpm-3 (deployment): created
 --> etcd-kpm-1 (service): created
 --> etcd-kpm-2 (service): created
 --> etcd-kpm-3 (service): created
 --> etcd (service): created

 02 - coreos/kpm-registry:
 --> kpm (namespace): ok
 --> kpm-registry (deployment): created
 --> kpm-registry (service): created

```

Voila! Now you can tear down the bootstrapping registry if you'd like, e.g. stop the docker container and the API server as run by gunicorn.

