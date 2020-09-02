---
layout: single
title: "Working with Istio Ingress"
categories: service-mesh istio
classes: wide
---

> Coming from a low-layer Networking background I've always struggled a bit with Istio. Even something like getting Ingress setup baffled me for the longest time. Really though, it's no harder than configuring ADSL on a CISCO-827 and I did that enough times! Difference was that I never wrote down my experiences with networking here I hope to capture an easy way to get Istio Ingress configured on your dev environment

## Prerequisites

You'll need a Kubernetes cluster to test this. If you're comfortable with getting that setup you can use minikube or similar. Otherwise, I have a [how-to guide](http://localhost:4000/container-orchestration/kubernetes/headless-minikube/) on how to set up minikube on a dedicated development machine.

## Install the Istio Operator

Since the release of v1.6, Istio can now be installed using an Operator.

> Instead of manually installing, upgrading, and uninstalling Istio in a production environment, you can instead let the Istio operator manage the installation for you. This relieves you of the burden of managing different istioctl versions. Simply update the operator custom resource (CR) and the operator controller will apply the corresponding configuration changes for you.

Sounds pretty good right? Well it did to me so I thought I'd give it a try.

{% highlight bash %}

## Download the latest Istioctl
$ curl -sL https://istio.io/downloadIstioctl | sh -

## Add the Istioctl binary to your $PATH
$ export PATH=$PATH:$HOME/.istioctl/bin

## (Optional) Connect to your Minikube Cluster
$ kubectl config use-context minikube
Switched to context "minikube".

## Install the Istio Operator
$ istioctl operator init
Using operator Deployment image: docker.io/istio/operator:1.7.0
✔ Istio operator installed
✔ Installation complete

{% endhighlight %}


Well that was pretty straight forward - except that's just getting the Operator installed

## Installing Istio

Firstly it's important to decide what your use case for Istio will be - this will determine your config profile. More on that from the [offical site](https://istio.io/latest/docs/setup/additional-setup/config-profiles/). I'm wanting to tinker with Istio and also write some more demonstrations so I went with the **demo** profile but could have gone with **default** for the purposes of this demonstration.

{% highlight bash %}

## Create the Istio System Namespace
$ kubectl create ns istio-system

## Tell the IstioOperator to install Istio with the `demo` config profile
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
EOF

{% endhighlight %}

Installation was extremely quick and I was able to verify that the installation went in OK but running the following commands

{% highlight bash %}

## Check that Istio System Services are online
$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.104.91.191   <none>        80/TCP,443/TCP,15443/TCP                                                     2m24s
istio-ingressgateway   LoadBalancer   10.102.57.20    <pending>     15021:30465/TCP,80:30751/TCP,443:30765/TCP,31400:30467/TCP,15443:32441/TCP   2m24s
istiod                 ClusterIP      10.99.144.219   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                2m44s

## Check that Istio System Pods are online
$ k get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-695f5944d8-bz7ls    1/1     Running   0          2m54s
istio-ingressgateway-5c697d4cd7-tl2f9   1/1     Running   0          2m54s
istiod-59747cbfdd-x285g                 1/1     Running   0          3m14s

{% endhighlight %}

## Bonus steps

I was a little underwhelmed with the installation process so I thought I'd up the stakes and change from the **demo** config profile to the **default** profile. Mainly because I wanted to give it a try but also because I don't really need an egress gateway for now.

{% highlight bash %}

## Switching the config profile from `demo` to `default`
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
EOF

## Check that there's been a change in Istio-System Services
$ k get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                      AGE
istio-ingressgateway   LoadBalancer   10.102.57.20    <pending>     15021:32067/TCP,80:31349/TCP,443:30210/TCP,15443:30790/TCP   28m
istiod                 ClusterIP      10.99.144.219   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                28m

## Check that there's been a change in Istio-System Pods
$ k get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-658c5c4489-n5nqb   1/1     Running   0          19s
istiod-6f5fd7cb8f-wxhf7                 1/1     Running   0          23s

{% endhighlight %}

... and with that the Istio Egress Gateway was gone. Pretty nifty!

## Deploy an Application

Deploying the application to the minikube cluster is a fairly straight forward process. Here I'm using [httpbin](https://httpbin.org/) but any application that exposes a http end-point will do.

{% highlight bash %}

## Create a Namespace, Service and Deployment
$kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  labels:
    app: httpbin
  namespace: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - image: kennethreitz/httpbin:latest
        imagePullPolicy: Always
        name: httpbin
        ports:
        - containerPort: 80
          protocol: TCP
EOF

## Check that the httpbin service is online
$ kubectl get svc -n httpbin
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
httpbin   ClusterIP   10.106.205.23   <none>        8080/TCP   10m

## Check that the httpbin pod is online
$ kubectl get pods -n httpbin
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-558f4b785d-4qwxw   1/1     Running   0          8m40s

{% endhighlight %}

## Make the Application accessible to the outside world

... well to your laptop anyways.

Here's the bit that stumped me about Istio Ingress for the longest time. What's all this Virtual Service nonsense anyways? Well rather than read how about a picture of how it works?
