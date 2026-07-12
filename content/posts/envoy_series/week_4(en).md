---
title: "Contributing to Envoy Proxy #4: Building Envoy Proxy"
date: 2026-07-12T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 5
ShowToc: true
TocOpen: true
---
{{< series >}}

# Overview

In the previous post, we pulled the build image and brought up its container. With the build environment ready, this time we will actually build Envoy Proxy inside that container, then bring up Redis and Envoy Proxy on Minikube to see the whole stack running end to end.

# Building Envoy

Building Envoy Proxy is — surprisingly — a one-liner. Bazel handles the entire build:

```shell
$ cd /source/source/
$ bazel build //source/exe:envoy-static -c dbg
```

This only works if you have already pulled the `envoy-build-ubuntu` Docker image we covered in the [previous post]({{< ref "posts/envoy_series/week_3(en).md#pulling-the-envoy-build-image" >}}). If you have not, follow that guide and set the container up before continuing.

![Envoy Build](/envoy_series_build_envoy.png)

For reference, here is roughly how long the build takes on different machines:

- **Mac M1 Max (32 GB RAM):** about 3 hours 30 minutes
- **Mac M4 Pro (48 GB RAM):** about 2 hours
- **Mac M5 Pro (64 GB RAM, current laptop):** about 4,619 seconds

![Envoy Build Success](/envoy_series_build_envoy_success_time.png)

If your machine has enough resources, a single run of the command above is all you need. Honestly, though, Envoy is enormous to build, and a single-shot build will not fit into memory on most machines. When that happens, you need to throttle the resources the build is allowed to use. We already emphasized this in an earlier post — remember?

> Before reading this section, please complete the [Docker setup guide]({{< ref "posts/envoy_series/week_1(en).md#3-docker" >}}) from Part 1 — install Docker Desktop and adjust its resource limits.

Based on my own experience, you need **at least 30 GB of RAM** for a stable single-shot build. There is no hard rule written down anywhere; if the build fails outright, just assume your machine cannot hold it in memory at once. In that case, the move is to limit CPU and memory explicitly via `--local_resources`.

The example below uses `0.4` of the host resources. The smaller you set this, the longer the build takes, so tune it to whatever your current laptop can spare. On my M1 Max, I allocated 24 GB of RAM to Docker and throttled both RAM and CPU to around `0.8`.

```shell
$ bazel build //source/exe:envoy-static -c dbg \
    --local_resources=cpu=HOST_CPUS*.4 \
    --local_resources=memory=HOST_RAM*.4
```

Once the build finally finishes, the Envoy Proxy binary lands at the following path inside the container:

```
/source/source/bazel-bin/source/exe/envoy-static
```

The binary is huge, as you will see from the file size.

![Envoy Build Success](/envoy_series_build_envoy_success_binary.png)

Copy it back to the host:

```shell
$ cd ~/Desktop/cncf/
$ docker cp b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2:/source/source/bazel-bin/source/exe/envoy-static .
```

# Bringing Up the Infrastructure

We are not going to run the binary we just built today. Instead, we will bring up Redis and Envoy separately, and swap in our locally built Envoy in the next post.

Start the Kubernetes cluster (Minikube):

```shell
$ minikube start
```

![Start Minikube](/envoy_series_minikube_start.png)

> **Prerequisite:** Before proceeding, follow the [Minikube installation guide]({{< ref "posts/envoy_series/week_2(en).md#installing-minikube" >}}) from Part 2 and make sure both `minikube` and `kubectl` are installed.

After the cluster is up, open a **separate** terminal for Minikube control and run the command below. This is intentional: the Envoy build environment is *not* a container inside the cluster — it is fully isolated from the Minikube environment where the actual services (Redis, Envoy) will run. Separating the two terminals makes that distinction obvious.

```shell
eval $(minikube docker-env)
```

With the cluster ready, clone the example repo and apply the YAML files under its `basic` directory:

```shell
git clone https://github.com/qerogram/example-envoy-redis
cd example-envoy-redis/basic

kubectl apply -f envoy-k8s.yaml
kubectl apply -f redis-k8s.yaml

kubectl get all
```

![Start Minikube](/envoy_series_minikube_deploy_basic.png)

For Envoy Proxy, pull the image once so the Pod starts immediately:

```shell
docker pull envoyproxy/envoy:v1.38-latest
```

## Testing Redis Directly

Now let's verify everything works end to end. First, install `redis-cli`:

- **macOS:** `brew install redis`
- **Windows:** Download a release from the [Microsoft Archive Redis](https://github.com/microsoftarchive/redis/releases) page.

Before connecting to the Pod, we need to port-forward. In the terminal where you ran `eval $(minikube docker-env)`, run:

```shell
kubectl port-forward deployments/redis 6379:6379
```

Open a second terminal and connect:

```shell
$ redis-cli -p 6379
127.0.0.1:6379> hello
```

![Start Minikube](/envoy_series_minikube_test_redis.png)

## Testing Redis Through Envoy Proxy

Next, let's reach Redis *through* Envoy Proxy. Forward port 16379 instead:

```shell
kubectl port-forward deployments/envoy 16379:16379
```

Open another terminal and connect:

```shell
$ redis-cli -p 16379
127.0.0.1:16379> hello
```

![Start Minikube](/envoy_series_minikube_test_envoy.png)

# Wrapping Up

In this post, we built Envoy Proxy from source and brought up Redis plus Envoy on a Minikube cluster, then verified connectivity both directly and through Envoy. In the next post, we will swap in the locally built binary and run the same flow against it.

See you in the next article.