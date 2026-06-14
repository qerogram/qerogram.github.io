---
title: "[EN] Contributing to Envoy Proxy #2: Setting Up the Build Environment — Part 1"
date: 2026-06-14T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"
series_order: 3
ShowToc: true
TocOpen: true
---
{{< series >}}

# Overview

This post walks through bringing up a local Kubernetes cluster. Managed offerings such as EKS, GKE, and AKS are the obvious choice for production workloads, but the recurring cost makes them impractical for everyday experimentation. As a more economical alternative, we will stand up a single-node cluster on a personal machine — a setup that is more than sufficient for the kind of iteration we need.

Once the cluster is up, we will install the two tools you will reach for most often: **`kubectl`**, the official command-line client, and **`k9s`**, a terminal UI that turns routine cluster operations into a far more pleasant experience.

# Installing Minikube

We will start with Minikube. The official installation instructions are kept current on the [Minikube SIG site](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download) — pick the path that matches your platform. For a guided walkthrough, the Kubernetes project also publishes a beginner-friendly [Hello Minikube](https://kubernetes.io/ko/docs/tutorials/hello-minikube/) tutorial that is worth bookmarking.

I am on Apple Silicon, so I went with Homebrew — by far the lowest-friction option on macOS. The relevant instructions are shown below.

![Minikube SIG install instructions](/envoy_series_minikube_install.png)

In short, installation reduces to a single command:

```shell
brew install minikube
```

# Bringing Up the Cluster

With Minikube installed, the following command provisions a local Kubernetes cluster:

```shell
minikube start
```

For this to succeed, **Docker must be running** on your machine — it is Minikube's default container runtime, and the command above delegates all container provisioning to it.

If Docker is unavailable, or you simply prefer not to use it, Minikube also supports alternative drivers such as **Podman** and **Rancher Desktop**. The full list of supported drivers is documented in the [Minikube driver reference](https://minikube.sigs.k8s.io/docs/drivers/). To select one explicitly, pass it via `--driver`. For example, to use Podman:

```shell
minikube start --driver=podman
```

![Minikube start](/envoy_series_minikube_start.png)

# Operating the Cluster

`kubectl` is the standard client for interacting with a Kubernetes API server. Official installation guides for Linux, macOS, and Windows are available in the Kubernetes documentation:

- [Installing kubectl](https://kubernetes.io/ko/docs/tasks/tools/)

<br/>

For day-to-day usage, the reference below is the canonical resource:

- [kubectl reference](https://kubernetes.io/ko/docs/reference/kubectl/)

Once everything is in place, verify the connection by listing resources in the default namespace:

```shell

❯ kubectl get all
NAME                         READY   STATUS    RESTARTS        AGE
pod/envoy-576d5d575-57zqn    1/1     Running   4 (5d20h ago)   13d
pod/redis-748cffd85d-x7ssz   1/1     Running   3 (5d20h ago)   13d

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/envoy-service   ClusterIP   10.104.133.242   <none>        16379/TCP   13d
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP     13d
service/redis-service   ClusterIP   10.106.46.93     <none>        6379/TCP    13d

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/envoy   1/1     1            1           13d
deployment.apps/redis   1/1     1            1           13d

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/envoy-576d5d575    1         1         1         13d
replicaset.apps/redis-748cffd85d   1         1         1         13d
```

> **Note:** The output above reflects my own cluster, which still holds the `envoy` and `redis` workloads from earlier experiments. On a fresh setup, the only entry you should expect to see is `service/kubernetes` — if that line is present, your cluster is healthy and you are good to go.

# Wrapping Up

With Minikube and `kubectl` in place, the foundation for our local test environment is ready. In the next post, we will pull the Envoy build image from Docker Hub and run our first build inside a container.

See you in the next article.
