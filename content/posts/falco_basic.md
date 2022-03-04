---
title: "Kubernetes security using Falco"
description: "Introdution to Falco runtime threat detection engine"
toc: true
authors: ["umangach"]
tags: ["kubernetes","falco","security"]
categories: []
series: []
date: 2022-03-04T12:35:19+05:30
lastmod: 2022-03-04T12:35:19+05:30
featuredVideo:
featuredImage:
draft: false
---
Falco is a runtime threat detection engine with custom rules and alerting capabilities. It becomes more powerful when combined with Falcosidekick, an automated threat response engine for Falco.
<!--more-->

Security focused static analysis tools capable of analyzing Kubernetes manifests help detect security misconfigurations early.
But, these checks are not really helpful for detecting threats and enforcing security at runtime. This is where Falco really shines.

In this post, we'll deploy Falco on minikube and see its threat detection engine in action.

## Scenarios

We will simulate following malicious actvities:
- Container running an interactive shell
- Write to directory holding system binaries

## Deploying Falco on Kubernetes

Falco's support for container specific context makes it really useful when using containers. However, it works on any Linux host.

Falco is a userspace application and uses a kernel module to intercept system calls. This architecture allows Falco userspace application to be integrated with external systems while keeping the driver separate from it. However, it is recommended to run Falco directly on the host to protect it from malicious activity.

## Try yourself

### Step 1 — Create a Kubernetes cluster

Use minikube with virtualbox driver to start a single node Kubernetes cluster.

```shell=
minikube start --driver=virtualbox
```

### Step 2 — Install Falco using helm

Install helm on the host and use it to install Falco in the minikube cluster.

```shell=
## Add the stable chart to Helm repository
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

## Install Falco using Helm
helm install falco falcosecurity/falco
```

### Step 3 - Check logs to ensure Falco is running

```shell=
kubectl logs -l app=falco -f
* Running dkms build failed, couldn't find /var/lib/dkms/falco/319368f1ad778691164d33d59945e00c5752cd27/build/make.log (with GCC /usr/bin/gcc-5)
* Trying to load a system falco module, if present
* Success: falco module found and loaded with modprobe
Wed Mar  2 07:42:15 2022: Falco version 0.31.0 (driver version 319368f1ad778691164d33d59945e00c5752cd27)
Wed Mar  2 07:42:15 2022: Falco initialized with configuration file /etc/falco/falco.yaml
Wed Mar  2 07:42:15 2022: Loading rules from file /etc/falco/falco_rules.yaml:
Wed Mar  2 07:42:15 2022: Loading rules from file /etc/falco/falco_rules.local.yaml:
Wed Mar  2 07:42:16 2022: Starting internal webserver, listening on port 8765
```

### Step 4 - Simulate malicious activity

Install an application and use it to simulate some malicious activities. We're using nginx here but any other application will work.

- Install `nginx`

```shell=
kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
falco-w62db              1/1     Running   0          8m40s
nginx-85b98978db-4b9pm   1/1     Running   0          16s
```

- Start an interactive shell to `nginx` pod

```shell=
kubectl exec -ti nginx-85b98978db-4b9pm -- /bin/bash
```

- Create a file in `/usr/local/bin` inside nginx pod

```shell=
root@nginx-85b98978db-4b9pm:/# cd /usr/local/bin/
root@nginx-85b98978db-4b9pm:/usr/local/bin# touch testfile
```

### Step 5 - Check Falco logs for rules violations

```shell=
kubectl logs -l app=falco -f
07:51:46.034248504: Notice A shell was spawned in a container with an attached terminal
(user=root user_loginuid=-1 k8s.ns=default k8s.pod=nginx-85b98978db-4b9pm container=c27c85e67186
shell=bash parent=runc cmdline=bash terminal=34816 container_id=c27c85e67186 image=nginx)
k8s.ns=default k8s.pod=nginx-85b98978db-4b9pm container=c27c85e67186

07:54:46.990180410: Error File below a monitored directory opened for writing
(user=root user_loginuid=-1 command=touch testfile file=/usr/local/bin/testfile
parent=bash pcmdline=bash gparent=<NA> container_id=c27c85e67186 image=nginx)
k8s.ns=default k8s.pod=nginx-85b98978db-4b9pm container=c27c85e67186 k8s.ns=default
k8s.pod=nginx-85b98978db-4b9pm container=c27c85e67186
```

### Step 6 - Uninstall Falco

```shell=
helm uninstall falco
release "falco" uninstalled
```

## Result

We were able to install Falco on minikube and test some default rules and alerts on violations.

## Next Steps

In this post, we used default configurations and rules for Falco.

In next posts, we will add some custom configurations and rules. We will also deploy Falcosidekick and see automated response in action.
