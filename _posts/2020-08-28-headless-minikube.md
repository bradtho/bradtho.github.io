---
layout: single
title: "Building a Kubernetes Dev Environment on a Headless Ubuntu 20.04 Server"
categories: container-orchestration kubernetes
classes: wide
---

> I've been needing a dedicated Kubernetes Dev Environment for the longest time - Minkube is great for this. However, the only piece of kit I have available and that's servicable is an old Dell Optiplex. I really didn't want to introduce another screen to my already busy desk so decided that whatever I was going to use for my Dev Environment had to be capable of being *Remote* (remote being under my desk) and *Headless* (command-line or browser only).
{: .notice--success}

## Prerequisites

So firstly, I had to get some pre-requisites installed. Some were already installed because I'd been using the server but not everything was there. In case you're starting from a fresh OS installation I'll provide the steps to get everything required.

I'm also running Ubuntu so these steps will be specific to that OS - I'll provide links to the vendor pages as I go in case you're running a different OS. The differences are fairly minimal.

* [Install kubectl](https:##kubernetes.io/docs/tasks/tools/install-kubectl/) - you'll need to do this on your management device and the server that's going to be running your Minkube instance.

{% highlight bash %}

## Download the latest kubectl binary and make it executable
$ curl -LO "https:##storage.googleapis.com/kubernetes-release/release/$(curl -s https:##storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" \ 
    && chmod +x ./kubectl

## Move the binary to somewhere in your $PATH
$ sudo mv kubectl /usr/local/bin/kubectl

## Make sure it installed correctly
$ kubectl version

{% endhighlight %}

* Install Virtualbox - you can use KVM or Docker but I went with Virtualbox for no reason in particular. To configure Virtualbox on the command-line you'll also need the Extension Pack.

{% highlight bash %}

## Download and Install Virtualbox
$ sudo apt install virtualbox virtualbox-ext-pack

{% endhighlight %}

* [Install Minikube](https:##kubernetes.io/docs/tasks/tools/install-minikube/)

{% highlight bash %}

## Download the minikube binary any make it executable
$ curl -Lo minikube https:##storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 

## Move the binary to somewhere in your $PATH
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube

## Make sure it installed correctly
$ minikube version

{% endhighlight %}

* Finally take a note of the IP Address of the remote server

> I'm assuming that you're SSHing into your headless server and therefore know its IP Address. If not you can run the following command.
{: .notice--success}

{% highlight bash %}

## Get the network hardware configuration information
$ ip a

## If you know the hardware address of your adapter you can pipe and grep
$ ip a | grep eno1
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000 inet 10.10.10.38/24 brd 10.10.10.255 scope global dynamic eno1

{% endhighlight %}

## Getting it all setup

Now we have all the required components and information needed we can start getting to the actual remote configuration setup.

So there are two key steps to getting this working:

1. Copying the minikube certificates to your local machine
2. Creating a Port Forwarding rule in your VirtualBox configuration to forward traffic to the kubectl port :8443

> NOTE: This can probably be replicated for 80 and 443 should you want to get the Kubernetes Dashboard running or in fact any other service you'd like to play around with.
{: .notice--success}

* Start minikube using the Virtualbox driver and setting the API Address to the IP address you took a not of earlier.

{% highlight bash %}

## Make VirtualBox the default driver
$ minikube config set driver virtualbox
❗  These changes will take effect upon a minikube delete and then a minikube start

## Start minikube with the apiserver-ips=<your host ip>
$ minikube start --apiserver-ips=10.10.10.38
😄  minikube v1.12.3 on Ubuntu 20.04
✨  Using the virtualbox driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🔥  Creating virtualbox VM (CPUs=2, Memory=3900MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.18.3 on Docker 19.03.12 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube"

{% endhighlight %}

* Now we have minikube running we need to copy the three certificates used to access the cluster over to your local machine - feel free to do this however you like such as encoding them into base64 but I just ran an *scp* from my local machine to pull them into the location I wanted them in.

{% highlight bash %}

## Run these from your local machine
$ scp user@server:~/.minikube/profiles/minikube/client.key ~/.kube/client.key
$ scp user@server:~/.minikube/profiles/minikube/client.crt ~/.kube/client.crt
$ scp user@server:~/.minikube/ca.crt ~/.kube/ca.crt

{% endhighlight %}

* Almost there! Now this is the part that really sets this apart from doing this on a machine with a GUI. We need to configure a Port Forwarding rule on the minikube virtual machine using VirtualBox's tool VBoxManage.

{% highlight zsh %}

## Get the minikube VM's network information
$ VBoxManage showvminfo minikube | grep 'NIC 1 Rule(0)'
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = 127.0.0.1, host port = 36235, guest ip = , guest port = 22

## Check that the port you're about to set isn't in use - I'm going to use 43910
$ sudo lsof -i -P -n

## Assuming the minikube vm is running set up the port forwarding rule with this command
$ VBoxManage controlvm minikube natpf1 "kubectl,tcp,,43910,,8443"

## If you minikube vm is not running you can run this command
$ VBoxManage modifyvm minikube natpf1 "kubectl,tcp,,43910,,8443"

{% endhighlight %}

* Nearly done! We now need to setup our kubeconfig file to use the certificates and API address we configured above.

{% highlight yaml %}

apiVersion: v1
clusters:
  - cluster:
    certificate-authority: ~/.kube/ca.crt
    server: https://10.10.10.38:43910
name: minikube
contexts:
  - context:
    cluster: minikube
    user: minikube
name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
  - name: minikube
user:
  client-certificate: ~/.kube/client.crt
  client-key: ~/.kube/client.key

{% endhighlight %}

* Finally let's test if we can connect to the remote instance of minikube.

{% highlight bash %}

$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   25h   v1.18.3

{% endhighlight %}

* Thanks for reading!
