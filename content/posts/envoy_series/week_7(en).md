---
title: "[EN] Contributing to Envoy Proxy #7: Building and Verifying bitfield_ro"
date: 2026-08-14T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"
series_order: 8
ShowToc: true
TocOpen: true
---
{{< series >}}

# Background

In the [previous post (#6: Issue Analysis — Where do we add bitfield_ro?)]({{< ref "posts/envoy_series/week_6(en).md" >}}), we analyzed where the change should land and concluded that `bitfield_ro` belongs in `simpleCommands()`. In this post, we will redeploy the Envoy binary with that change into the Minikube cluster we set up in [week_5](/posts/envoy_series/week_5(en)/) and confirm that the new command works end-to-end.

---

# Preparation

The deployment flow is similar to [week_5](/posts/envoy_series/week_5(en)/): copy the locally built `envoy-static` into the `custom/` directory of the [qerogram-example-envoy-redis](https://github.com/qerogram/example-envoy-redis) repository and apply the prepared Kubernetes manifests.

First, copy the binary from the build container to the host.

> If you do not remember how to obtain `envoy-static`, refer to the [Envoy Proxy Build Guide]({{< ref "posts/envoy_series/week_4.md#build" >}}).
>
> ```shell
> $ cd ~/Desktop/cncf/
> $ docker cp b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2:/source/source/bazel-bin/source/exe/envoy-static .
> ```

The `Dockerfile` inside `custom/` packages the binary we built into a container image. In other words, you can think of it as **overwriting the official Envoy image with our own binary**.

> **Tip:** Images built with `docker build` are stored in the **host's Docker daemon**. Since Minikube uses its own Docker daemon, you must point the terminal's Docker context at Minikube before building — otherwise `kubectl apply -f` will fail with `ImagePullBackOff`.
>
> ```shell
> eval $(minikube docker-env)
> ```
>
> See the [infrastructure setup section of week_4]({{< ref "posts/envoy_series/week_4.md" >}}) for the background.

Once you are ready, run the following commands in order. First, delete the existing Envoy resources and remove the previously built Docker image.

```shell
cd ~/Desktop/cncf/
kubectl delete -f envoy-k8s.yaml
docker rmi custom-envoy:v1
```

---

# Redeploying the image

After the cleanup, rebuild the container image and re-apply the Kubernetes manifests.

```shell
# Assume envoy-static from the previous build is in the current directory.
git clone https://github.com/qerogram/example-envoy-redis

mv ./envoy-static example-envoy-redis/custom
cd example-envoy-redis/custom

# Build the custom Envoy image
docker build -t custom-envoy:v1 .

# Apply Kubernetes manifests
kubectl apply -f envoy-k8s.yaml
kubectl apply -f redis-k8s.yaml

# Verify deployment status
kubectl get all
```

> **Note:** `envoy-k8s.yaml` already references `custom-envoy:v1` instead of `envoyproxy/envoy:v1.38-latest`, so you do not need to change the image name separately.

Once the pods are running, open two terminals and start port-forwarding again.

```shell
kubectl port-forward deployments/redis 6379:6379
kubectl port-forward deployments/envoy 16379:16379
```

Before redeploying the new binary, the `bitfield_ro` command still fails when sent through Envoy.

![Before redeploying the updated Envoy binary](/envoy_series_redis_proxy_week7_1.png)

```shell
bitfield_ro qero get u8 0
```

After redeploying the binary with the change, the command works as expected:

![After redeploying the updated Envoy binary](/envoy_series_redis_proxy_week7_2.png)

---

# Coverage test

We can confirm that `bitfield_ro` is now recognized and forwarded correctly. Because the change only adds the command to `simpleCommands()`, the existing command-splitter test already exercises this registration path, so no separate test is needed.

Before wrapping up today's post, let us also check whether the change meets the project's coverage requirement.

Return to the container used to build Envoy, then run the command below. It reports how much of the source code is covered by the current tests.

> Side note: running the coverage test can invalidate the Bazel cache, so subsequent builds may take a bit longer.

```shell
cd /source/source
./ci/do_ci.sh coverage //test/extensions/filters/network/redis_proxy/...
```

![Coverage Test 1](/envoy_series_redis_proxy_week7_3.png)

You should see output like the image below. For Redis Proxy, the maintainers require coverage above **96.6%**. If coverage falls below that threshold, the PR's CI check will fail, so you will need to add tests or make other changes to raise coverage before merging.

![Coverage Test 2](/envoy_series_redis_proxy_week7_4.png)

Our result is **97.2%**, so the coverage requirement is met.

The coverage threshold is defined in `test/coverage.yaml`. If your filter has no specific threshold, the default target is **96.6%**.

![Coverage Check](/envoy_series_redis_proxy_week7_5.png)

At this point, you can package your custom Envoy in a container image and publish it to a private container registry for internal or personal use.

---

# Next post preview

Starting in the next post, we will walk through the process of submitting a Pull Request. Once the change is merged upstream, you will no longer need a private registry; you can use the official upstream release instead.

fin.
