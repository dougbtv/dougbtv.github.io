---
author: dougbtv
comments: true
date: 2017-02-09 15:10:01-05:00
layout: post
slug: kubernetes-on-centos
title: Let's (manually) run k8s on CentOS!
category: nfvpe
---

So sometimes it's handy to have a plain-old-Kubernetes running on CentOS 7. Either for development purposes, or to check out something new. Our goal today is to install Kubernetes by hand on a small cluster of 3 CentOS 7 boxen. We'll spin up some libvirt VMs running CentOS generic cloud images, get Kubernetes spun up on those, and then we'll run a test pod to prove it works. Also, this gives you some exposure to some of the components that are running 'under the hood'.

Let's follow [the official Kubernetes guide for CentOS](https://kubernetes.io/docs/getting-started-guides/centos/centos_manual_config/) to get us started.

But, before that, we'll need some VMs to use as the basis of our three machine cluster.

## Let's spin up a couple VM's

So, we're going to assume you have a machine with libvirt to spin up some VMs. In this case I'm going to use a [CentOS Cloud Image](https://cloud.centos.org/centos/7/images/), and I'm going to spin them up in this novel way using [this guide to spin those up easily](http://giovannitorres.me/create-a-linux-lab-on-kvm-using-cloud-images.html).

So let's make sure we have the prerequisites. Firstly, I am using Fedora 25 as my workstation, and I'm going to spin up the machines there.

```
$ sudo dnf install libvirt-client virt-install genisoimage
```

I have a directory called `/home/vms` and I'm going to put everything there (this basic qcow2 cloud image, and my virtual machine disk images), so let's make sure we download the cloud image there, too.

```
# In case you need somewhere to store your VM "things"
$ mkdir /home/vms

# Download the image
$ cd /home/vms/
$ wget -O /home/vms/CentOS-7-x86_64-GenericCloud.qcow2.xz https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1612.qcow2.xz

# Extract the downloaded image...
$ xz -d CentOS-7-x86_64-GenericCloud.qcow2.xz
```

I originally had this in the wrong place, so just make sure the image winds up in the right place, it should be @ `/home/vms/CentOS-7-x86_64-GenericCloud.qcow2`.


Now let's download [the gist for spinning up a cloud image in libvirt](https://gist.github.com/giovtorres/0049cec554179d96e0a8329930a6d724) and we'll change it's mode so we can execute it.

```
# Download the Gist
$ wget -O spin-up-generic.sh https://gist.githubusercontent.com/giovtorres/0049cec554179d96e0a8329930a6d724/raw/f7520fbbf1e4a54f898cf8cc51e3eaac9167f178/virt-install-centos

# Make it executable
$ chmod 0755 spin-up-generic.sh 

# Change the default image directory to the one we created earlier.
$ sed -i -e 's|~/virt/images|/home/vms|g' spin-up-generic.sh
```

But, wait! There's more. Go ahead and make sure you have an SSH public key you can add to the `spin-up-generic.sh` script. Make sure you cat the appropriate public key.

```
# Chuck your ssh public key into a variable...
$ sshpub=$(cat ~/.ssh/id_rsa.pub)

# Sed the file and replace the dummy public key with your own
# (You could also edit the file by hand and do a find for "ssh-rsa")
$ sed -i -e "/ssh-rsa/c\  - $sshpub" spin-up-generic.sh
```

Now, we can spin up a few VMs, we're going to spin up a master and 2 minions. You'll note that you get an IP address from this script for each machine, take note of those cause we'll need it in the next steps. Depending on your setup for libvirt you might have to use sudo. 

```
[root@yoda vms]# ./spin-up-generic.sh centos-master
[...]
Wed, 08 Feb 2017 16:28:21 -0500 DONE. SSH to centos-master using 192.168.122.21 with  username 'centos'.

[root@yoda vms]# ./spin-up-generic.sh centos-minion-1
[...]
Wed, 08 Feb 2017 16:28:49 -0500 DONE. SSH to centos-minion-1 using 192.168.122.18 with  username 'centos'.

[root@yoda vms]# ./spin-up-generic.sh centos-minion-2
[...]
Wed, 08 Feb 2017 16:29:16 -0500 DONE. SSH to centos-minion-2 using 192.168.122.208 with  username 'centos'.
```

Alright, now you should be able to SSH to these guys, ssh into the master node to test it out...

```
$ ssh centos@192.168.122.21
```

## Let's start installing k8s!

Alrighty, so there's things we're going to want to do across multiple hosts. Since the goal here is to do this manually (e.g. not creating an ansible playbook) we're going to have a few for loops to do this stuff efficiently for us. So, set a variable with the class D octet from each of the IPs above. (And one for the master & the minions, too, we'll use this later.)

```
class_d="21 18 208"
master_ip="192.168.122.21"
minion_ips="192.168.122.18 192.168.122.208"
```

And for a test, just go and run this...

```
$ for i in $class_d; do ssh centos@192.168.122.$i 'cat /etc/redhat-release'; done
```

You may have to accept the key finger print for each box.

### Install Kubernetes RPM requirements

Now we're creating some repo files for the k8s components.

```
$ for i in $class_d; do ssh centos@192.168.122.$i 'echo "[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0
" | sudo tee /etc/yum.repos.d/virt7-docker-common-release.repo'; done
```

Now install etcd, kubernetes & flannel on all the boxen.

```
$ for i in $class_d; do ssh centos@192.168.122.$i 'sudo yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel'; done
```


### Setup /etc/hosts

Now, we need to add to our hosts files the hostnames for each of these three files, so let's mock up the lines we want to add, in my case, the lines I'll add look like:

```
192.168.122.21 centos-master
192.168.122.18 centos-minion-1
192.168.122.208 centos-minion-2
```

So I'll append using tee in a loop like:

```
for i in $class_d; do ssh centos@192.168.122.$i 'echo "
192.168.122.21 centos-master
192.168.122.18 centos-minion-1
192.168.122.208 centos-minion-2" | sudo tee -a /etc/hosts'; done
```

### Setup Kubernetes configuration

Now we're going to chuck in a `/etc/kubernetes/config` file, same across all boxes. So let's make a local version of it and scp it. I tried to do it in one command, but, too much trickery between looping SSH and heredocs and what not. So, make this file...

```
cat << EOF > ./kubernetes.config
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://centos-master:2379"

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the replication controller and scheduler find the kube-apiserver
KUBE_MASTER="--master=http://centos-master:8080"
EOF
```

Now scp it to all the hosts...

```
for i in $class_d; do scp ./kubernetes.config centos@192.168.122.$i:~/kubernetes.config; done
```

And finally move it into place.

```
for i in $class_d; do ssh centos@192.168.122.$i 'sudo mv /home/centos/kubernetes.config /etc/kubernetes/config'; done
```

### Wave goodbye to your security

So the official docs do things that generally... I'd say "Don't do that.", but, alas, we're going with the official docs, and this likely simplifies some things. So, while we're here we're going to follow those instructions, and we're going to `setenforce 0` and then disable the firewalls.

```
for i in $class_d; do ssh centos@192.168.122.$i 'sudo setenforce 0; sudo systemctl disable iptables-services firewalld; sudo systemctl stop iptables-services firewalld; echo'; done
```

## Configure Kube services on the master

Here we setup etcd on the master...

```
ssh centos@$master_ip 'sudo /bin/bash -c "
cat << EOF > /etc/etcd/etcd.conf
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

#[cluster]
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
EOF
"'
```

And the etcd api server...

```
ssh centos@$master_ip 'sudo /bin/bash -c "
cat << EOF > /etc/kubernetes/apiserver
# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port kubelets listen on
KUBELET_PORT="--kubelet-port=10250"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# Add your own!
KUBE_API_ARGS=""
EOF
"'
```

And we start etcd and specify some keys, remember from the docs:

> Warning This network must be unused in your network infrastructure! 172.30.0.0/16 is free in our network.

So go ahead and start that add the keys assuming that warning is OK...

```
ssh centos@$master_ip 'sudo systemctl start etcd; sudo etcdctl mkdir /kube-centos/network; sudo etcdctl mk /kube-centos/network/config "{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"'
```

If you'd like to check that etcd key, you can do:

```
ssh centos@$master_ip 'etcdctl get /kube-centos/network/config'
```

Now, configure flannel... (later we'll do this on the nodes as well)

```
ssh centos@$master_ip 'sudo /bin/bash -c "
cat << EOF > /etc/sysconfig/flanneld
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://centos-master:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
EOF
"'
```

And then restart and enable the services we need...


```
ssh centos@$master_ip 'sudo /bin/bash -c "
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler flanneld; do
    systemctl restart \$SERVICES
    systemctl enable \$SERVICES
    systemctl status \$SERVICES
done
"'
```

## Mildly verifying the services on the master

There's a lot going on above, right? I, in fact, made a few mistakes while performing the above actions. I had a typo. So, let's make sure the services are active.

```
ssh centos@$master_ip 'sudo /bin/bash -c "
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler flanneld; do
    systemctl status \$SERVICES | grep -P \"(\.service \-|Active)\"
done
"'
```

Make sure each entry this is "Active" state of "active." If for some reason one isn't, go and check the journald logs, on the master, for it with:

```
journalctl -f -u kube-apiserver
```

(Naturally replacing the service name with the one in trouble from above.)

## Configure the minion nodes

Ok, first thing we're going to manually set each of the hostnames for the minions. Our VM spin up script names them "your_name.example.local", not quite good enough. So let's manually set each of those.

```
ssh centos@192.168.122.18 'sudo hostnamectl set-hostname centos-minion-1'
ssh centos@192.168.122.208 'sudo hostnamectl set-hostname centos-minion-2'
```

Now just double check those

```
for i in $minion_ips; do ssh centos@$i 'hostname'; done
```

Ok cool, that means we can simplify a few steps following.

Now we can go ahead and configure the kubelet.

```
for i in $minion_ips; do ssh centos@$i 'sudo /bin/bash -c "
cat << EOF > /etc/kubernetes/kubelet
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
# Check the node number!
# KUBELET_HOSTNAME="--hostname-override=centos-minion-n"

# Location of the api-server
KUBELET_API_SERVER="--api-servers=http://centos-master:8080"

# Add your own!
KUBELET_ARGS=""
EOF
"'; done
```

Now, setup flannel...

```
for i in $minion_ips; do ssh centos@$i 'sudo /bin/bash -c "
cat << EOF > /etc/sysconfig/flanneld
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://centos-master:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
EOF
"'; done
```

And get the services running....

```
for i in $minion_ips; do ssh centos@$i 'sudo /bin/bash -c "
for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl restart \$SERVICES
    systemctl enable \$SERVICES
    systemctl status \$SERVICES
done
"'; done
```

And we'll double check those

```
for i in $minion_ips; do ssh centos@$i 'sudo /bin/bash -c "
for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl status \$SERVICES | grep -P \"(\.service \-|Active)\"
done
"'; done
```

Great!

## Drum roll please.... Let's see if it's all running!

So OK, one more step... Let's set some default in kubectl, we'll do this from the master. In this case... Now I'm going to ssh directly to that machine and work from there...

```
$ ssh centos@192.168.122.21
```

And then we'll perform:

```
kubectl config set-cluster default-cluster --server=http://centos-master:8080
kubectl config set-context default-context --cluster=default-cluster --user=default-admin
kubectl config use-context default-context
```

Here's... the moment of truth. Let's see if we can see all the nodes...

```
[centos@centos-master ~]$ kubectl get nodes
NAME              STATUS    AGE
centos-minion-1   Ready     2m
centos-minion-2   Ready     2m
```

Yours should look about like the above!

## So, you wanna run a pod?

Well this isn't much fun without having a pod running, so let's at least get something running.

### Create an nginx pod

Let's create an nginx pod... Create a pod spec anywhere you want on the master, here's what mine looks like

```
[centos@centos-master ~]$ cat nginx_pod.yaml 
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

```

Now you can create given that yaml file.

```
[centos@centos-master ~]$ kubectl create -f nginx_pod.yaml 
```

And you can see it being create when you get pods...

```
[centos@centos-master ~]$ kubectl get pods
NAME          READY     STATUS              RESTARTS   AGE
nginx-8rajt   0/1       ContainerCreating   0          10s
nginx-w2yja   0/1       ContainerCreating   0          10s
```

And you can get details about the pod with:

```
[centos@centos-master ~]$ kubectl describe pod nginx-8rajt
Name:       nginx-8rajt
Namespace:  default
Node:       centos-minion-2/192.168.122.208
Start Time: Thu, 09 Feb 2017 19:39:14 +0000
Labels:     app=nginx
Status:     Pending
IP:     
```

In this case you can see this is running on `centos-minion-2`. And there's two instances of this pod! We specified `replicas: 2` in our pod spec. And that's the job of the kubelet -- make sure instances are running, and in this case, it's going to make sure 2 are running across our hosts.

### Create a service to expose nginx.

Now that's all well and good, but... What if we want to, y'know, serve something? (Omitting, uhhh, content!) But, we can do that by exposing this to a service.

So let's go and expose it... Let's create a service spec. Here's what mine looks like:

```
[centos@centos-master ~]$ cat nginx_service.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    name: nginxservice
  name: nginxservice
spec:
  ports:
    # The port that this service should run on.
    - port: 9090
  # Label keys and values that must match in order to receive traffic for this service.
  selector:
    app: nginx
  type: LoadBalancer
```

And then we create that...

```
[centos@centos-master ~]$ kubectl create -f nginx_service.yaml
service "nginxservice" created
```

And we can see what's running by getting the services and describing the service.

```
[centos@centos-master ~]$ kubectl get services
NAME           CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes     10.254.0.1       <none>        443/TCP    1h
nginxservice   10.254.110.163   <pending>     9090/TCP   58s

[centos@centos-master ~]$ kubectl describe service nginxservice
Name:           nginxservice
Namespace:      default
Labels:         name=nginxservice
Selector:       app=nginx
Type:           LoadBalancer
IP:         10.254.110.163
Port:           <unset> 9090/TCP
NodePort:       <unset> 32702/TCP
Endpoints:      172.30.16.2:9090,172.30.52.2:9090
Session Affinity:   None
No events.
```

Oh so you want to actually curl it? Next time :) Leaving you with a teaser for the following installments. Maybe next time we'll do this all with Ansible instead of these tedious ssh commands.