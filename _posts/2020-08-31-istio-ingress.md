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

{% highlight bash %}

## Download the latest Istioctl
$ curl -sL https://istio.io/downloadIstioctl | sh -

## Add the Istioctl binary to your $PATH
$ export PATH=$PATH:$HOME/.istioctl/bin

{% endhighlight %}

## Deploy an Application

Deploying the application is a fairly straight forward process.

{% highlight yaml %}

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

{% endhighlight %}
