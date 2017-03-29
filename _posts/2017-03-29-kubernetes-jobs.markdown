---
author: dougbtv
comments: true
date: 2017-03-29 16:43:00-04:00
layout: post
slug: kubernetes-jobs
title: You had ONE JOB -- A Kubernetes job.
category: nfvpe
---

Let's take a look at how Kubernetes jobs are crafted. I had been jamming some kind of work-around shell scripts in the entrypoint\* for some containers in the [vnf-asterisk](https://github.com/redhat-nfvpe/vnf-asterisk) project that [Leif](https://github.com/leifmadsen) and I have been working on. And that's not perfect when we can use Kubernetes [jobs](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/), or in their new parlance, "run to completion finite workloads" (I'll stick to calling them "jobs"). They're one-shot containers that do one thing, and then end (sort of like a "oneshot" of systemd units, at least how we'll use them today). I like the idea of using them to complete some service discovery for me when other pods are coming up. Today we'll fire up a pod, and spin up a job to discover that pod (by querying the API for info about it), and put info into etcd. Let's get the job done.

This post also exists as a [gist on github](https://gist.github.com/dougbtv/67589a7b3e443d1b4e2cdf05698f58ca) where you can grab some files from, which I'll probably reference a couple times.

\* Not everyone likes having a custom entrypoint shell script, some people consider it a bit... "Jacked up". But, personally I don't depending on circumstance. Sometimes I think it's a pragmatic solution. So it's not always bad -- it depends on the case. But, where we can break things up and into their particular places, it's a GoodThing(TM).

Let's try firing up a kubernetes job and see how it goes. We'll use the [k8s jobs documentation as a basis](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/). But, as you'll see we'll need a bit more help as 

## Some requirements.

You'll need a Kubernetes cluster up and running here. If you don't, you can [spin up k8s on centos with this blog article](http://dougbtv.com//nfvpe/2017/02/16/kubernetes-1.5-centos/).

An bit of an editorial is that.... Y'know... OpenShift Origin kind of makes some of these things a little easier compared to vanilla K8s, especially with manging permissions and all that good stuff for the different accounts. It's a little more cinched down in some ways (which you want in production), but, there's some great considerations with `oc` to handle some of what we have to look at in more fine-grained detail herein.

## Running etcd.


You can pick up the YAML for this from [the gist](https://gist.github.com/dougbtv/67589a7b3e443d1b4e2cdf05698f58ca), and it should be easy to fire up etcd with:

```
kubectl create -f etcd.yaml
```

Assuming you've setup DNS correctly to resolve from the master (get the DNS pod IP, and put it in your resolve.conf and search cluster.local -- also see `scratch.sh` in the gist), you can check that it works...

```bash
# Set the value of the key "message" to be "sup sup"
[centos@kube-master ~]$ curl -L -X PUT http://etcd-client.default.svc.cluster.local:2379/v2/keys/message -d value="sup sup"
{"action":"set","node":{"key":"/message","value":"sup sup","modifiedIndex":9,"createdIndex":9}}

# Now retrieve that value.
[centos@kube-master ~]$ curl -L http://etcd-client.default.svc.cluster.local:2379/v2/keys/message 
{"action":"get","node":{"key":"/message","value":"sup sup","modifiedIndex":9,"createdIndex":9}}
```

## Authenticating against the API.

At least to me -- etcd is easy. Jobs are easy. The hardest part was authenticating against the API. So, let's step through how that works quickly. It's not difficult, I just didn't have all the pieces together at first. 

The idea here is that Kubernetes puts the default service account's API token into a file in the container in the pod @ `/var/run/secrets/kubernetes.io/serviceaccount/token` for the default service account in the default namespace. We then present that to the API in our curl command.

But, It did get me to read about [the kubectl proxy command](https://kubernetes.io/docs/user-guide/kubectl/kubectl_proxy/), [service accounts](https://kubernetes.io/docs/user-guide/service-accounts/), and [accessing the cluster](https://kubernetes.io/docs/concepts/cluster-administration/access-cluster/). When really what I needed was just a bit of a tip from this [stackoverflow answer](http://stackoverflow.com/questions/30690186/how-do-i-access-the-kubernetes-api-from-within-a-pod-container).

First off, you can see that you have the api running with

```
kubectl get svc --all-namespaces | grep -i kubernetes
```

Great, if you see the line there that means you have the API running, and you'll also be able to access it with DNS, which makes things a little cleaner.

Now that you can see that, we can go and access it... let's run a pod.

```
kubectl run demo --image=centos:centos7 --command -- /bin/bash -c 'while :; do sleep 10; done'
```

Alright, now that you've got this running (look for it with `kubectl get pods`), we can enter that container and query the API.

Let's do that just prove it. 

```bash
[centos@kube-master ~]$ kubectl exec -it demo-1260169299-96zts -- /bin/bash

# Not enough columns for me...
[root@demo-1260169299-96zts /]# stty rows 50 cols 132

# Pull up the service account token.
[root@demo-1260169299-96zts /]# KUBE_TOKEN=$(</var/run/secrets/kubernetes.io/serviceaccount/token)

# Show it if you want.
[root@demo-1260169299-96zts /]#  echo $KUBE_TOKEN

# Now you can query the API
[root@demo-1260169299-96zts /]# curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://kubernetes.default.svc.cluster.local
```

## Now, we have one job to do...

And that's to create a job. So let's create a job and we'll put the IP address of this "demo pod" into etcd. In theory we'd use this with something else to discover where it's based.

We'll figure out the IP address of the pod by querying the API. If you'd like to dig in a little bit and get your feet with the Kube API, may I suggest [this article from TheNewStack on taking the API for a spin](https://thenewstack.io/taking-kubernetes-api-spin/).

Why not just query always the API? Well. You could do that too. But, in my case we're going to generally standardize around using etcd. In part because in the full use-case we're going to also store other metadata there that's not "just the IP address".

So, we can query the API directly to find out the fact we're looking for, so let's do that just to test out that our results are OK in the end. 

I'm going to cheat here and run this little test from the master (instead of inside a container), it should work if you're deploying using my playbooks.

```bash 
# Figure out the pod name
[centos@kube-master ~]$ podname=$(curl -s http://localhost:8080/api/v1/namespaces/default/pods | jq ".items[] .metadata.name" | grep -i demo | sed -e 's/"//g')
[centos@kube-master ~]$ echo $podname
demo-1260169299-96zts

# Now using the podname, we can figure out the IP
[centos@kube-master ~]$ podip=$(curl -s http://localhost:8080/api/v1/namespaces/default/pods/$podname | jq '.status.podIP' | sed -s 's/"//g')
[centos@kube-master ~]$ echo $podip
10.244.2.11
```

Alright, that having been proven out we can create a job to do this for us, now too.

Let's go and define a job YAML definition. You'll find this one borrows generally heavily from the documentation, but, mixes up some things -- especially it uses a customized `centos:centos7` image of mine that has `jq` installed in it, it's called `dougbtv/jq` and it's available on dockerhub.

Also available in the gist, here's the `job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hoover
spec:
  template:
    metadata:
      name: hoover
    spec:
      containers:
      - name: hoover
        image: dougbtv/jq
        command: ["/bin/bash"]
        args:
          - "-c"
          - >
            KUBE_TOKEN=$(</var/run/secrets/kubernetes.io/serviceaccount/token) &&
            podname=$(curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/namespaces/default/pods | jq '.items[] .metadata.name' | grep -i demo | sed -e 's/\"//g') && 
            podip=$(curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://kubernetes.default.svc.cluster.local/api/v1/namespaces/default/pods/$podname | jq '.status.podIP' | sed -s 's/\"//g') &&
            echo "the pod is @ $podip" &&
            curl -L -X PUT http://etcd-client.default.svc.cluster.local:2379/v2/keys/podip -d value="$podip"
      restartPolicy: Never
```

Let's create it.

```
[centos@kube-master ~]$ kubectl create -f job.yaml 
```

It's named "hoover" as it's kinda sucking up some info from the API to do something with it.

So look for it in the list of ALL pods.

```
[centos@kube-master ~]$ kubectl get pods --show-all
NAME                    READY     STATUS      RESTARTS   AGE
demo-1260169299-96zts   1/1       Running     0          1h
etcd0                   1/1       Running     0          12d
etcd1                   1/1       Running     0          12d
etcd2                   1/1       Running     0          12d
hoover-fkbjj            0/1       Completed   0          20s
```

Now we can see what the logs are from it. It'll tell us what the IP is.

```
[centos@kube-master ~]$ kubectl logs hoover-fkbjj
```

That all said and done... we can complete this by seeing that the value made it to etcd.

```
[centos@kube-master ~]$ curl -L http://etcd-client.default.svc.cluster.local:2379/v2/keys/podip
{"action":"get","node":{"key":"/podip","value":"10.244.2.11","modifiedIndex":10,"createdIndex":10}}
```

Voila!
