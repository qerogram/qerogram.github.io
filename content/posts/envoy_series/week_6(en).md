---
title: "[EN] Contributing to Envoy Proxy #6: Issue Analysis — Where do we add bitfield_ro?"
date: 2026-08-09T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 7
ShowToc: true
TocOpen: true
---
{{< series >}}

# Background

In the [previous post (#5: Custom Envoy Proxy Deployment)]({{< ref "posts/envoy_series/week_5(en).md" >}}), I deployed a self-built Envoy into a Minikube cluster and compared the responses from accessing Redis directly versus going through the Envoy proxy. We confirmed that the `bitfield_ro` command is rejected as an **unknown command** by the proxy.

In this post, we will analyze **where and how** to fix that problem inside the Envoy Redis Proxy codebase. The goal is to pin down the exact modification points — not to modify code yet.

---

# What is BITFIELD_RO?

`BITFIELD_RO` is the read-only variant of `BITFIELD`, added in Redis 6.0. While `BITFIELD` lets you read and write multiple bitfields in a single key through `GET` / `SET` / `INCRBY` subcommands, `BITFIELD_RO` only allows `GET`.

The key observation from a routing perspective is that **both commands require hashing only one key — the first argument.** That means Envoy Redis Proxy's existing "single key → single shard" routing logic is enough. We do not need a new handler that merges responses. This judgment is the key conclusion of the analysis.

---

# How commands are organized in Redis Proxy

## Module layout

Let us first map out where the Redis Proxy code lives in the codebase.

| Path | Role |
|---|---|
| `source/extensions/filters/network/redis_proxy/` | The Redis proxy network filter itself (proxy, command splitter, router, connection pool) |
| `source/extensions/filters/network/common/redis/` | RESP codec, Redis client, command classification — shared primitives |
| `source/extensions/clusters/redis/` | Redis-specific cluster type (topology-aware LB) |
| `api/envoy/extensions/filters/network/redis_proxy/v3/redis_proxy.proto` | Filter config schema |
| `test/extensions/filters/network/redis_proxy/` | Unit and integration tests |

## Request flow

Let us trace what happens when a client sends a `bitfield_ro` request.

1. `proxy_filter.cc` — `ProxyFilter::onData()` hands the incoming bytes to the RESP decoder (`decoder_->decode`)
2. Once decoding finishes, `ProxyFilter::onRespValue()` is called, which forwards the request via `splitter_.makeRequest()`
3. `command_splitter_impl.cc` — `InstanceImpl::makeRequest()` uses the command name to look up the corresponding handler in the **lookup table**
![supported_commands.h — the simpleCommands() list. The bitfield_ro entry is added right next to bitfield](/envoy_series_redis_proxy_1.png)

4. The handler hashes the key, picks an upstream pool from the router (`router_impl.cc`), and sends the request through the connection pool (`conn_pool_impl.cc`)

Any command that is not registered in this lookup table at step 3 immediately gets an error response. The `unknown command 'bitfield_ro'` error we saw comes exactly from here.

## How commands get registered

So who populates the lookup table? Command support is defined by `SupportedCommands` in `common/redis/supported_commands.h`, and the splitter reads that list at startup. Commands are divided into the following sets by routing strategy.
![command_splitter_impl.cc — the loop that iterates simpleCommands() and auto-registers handlers](/envoy_series_redis_proxy_2.png)

![command_splitter_impl.cc — the loop that iterates simpleCommands() and auto-registers handlers](/envoy_series_redis_proxy_3.png)

- `simpleCommands()` — commands that hash a single key and go to one shard (`get`, `set`, `bitfield`, …)
- `multiKeyCommands()` — multi-key commands (`del`, `mget`, `mset`, …)
- `evalCommands()` — `eval`, `evalsha`
- `hashMultipleSumResultCommands()` — commands that fan out to multiple shards and sum the results (`exists`, …)
- `ClusterScopeCommands()` — commands sent to every shard (`flushall`, `config`, …)
- `randomShardCommands()` — commands sent to a random shard (`randomkey`, …)
- `transactionCommands()` — transactional commands (`multi`, `exec`, …)
- `writeCommands()` — the set of write commands. `isReadCommand()` treats anything **not in this set** as a read command

And the command splitter, at startup, **iterates these sets and auto-registers handlers** for each entry.

```cpp
// source/extensions/filters/network/redis_proxy/command_splitter_impl.cc
for (const std::string& command : Common::Redis::SupportedCommands::simpleCommands()) {
  addHandler(scope, stat_prefix, command, latency_in_micros, simple_command_handler_);
}
```

![command_splitter_impl.cc — the loop that iterates simpleCommands() and auto-registers handlers](/envoy_series_redis_proxy_4.png)

> **Note:** Adding a new command therefore boils down to answering one question: "which set does this command belong to?" Once you put it in a set, the lookup table registration follows automatically.

---

# Where to touch for adding bitfield_ro

As analyzed above, `BITFIELD_RO` only needs single-key hashing, so `simpleCommands()` is the right answer. Let us go file by file.

## 1. supported_commands.h — the heart of the change

```diff
 // source/extensions/filters/network/common/redis/supported_commands.h
-        "bitcount", "bitfield", "bitpos", "decr", ...
+        "bitcount", "bitfield", "bitfield_ro", "bitpos", "decr", ...
```

![command_splitter_impl.cc — the loop that iterates simpleCommands() and auto-registers handlers](/envoy_series_redis_proxy_5.png)


> **Tip:** Since `BITFIELD_RO` is read-only, do **not** add it to `writeCommands()`. `isReadCommand()` is implemented as `!writeCommands().contains(command)`, so simply leaving it out is enough to be treated as a read command. This decision affects later read-only routing policies, such as directing reads to replicas.

## 2. command_splitter_impl.cc — no changes needed

The `InstanceImpl` constructor iterates `simpleCommands()` in full and registers each one with `simple_command_handler_`, so adding to the set is enough. A new handler is only required for commands that need special response splitting or merging, like `MGET` or `SCAN`.
![command_splitter_impl.cc — the loop that iterates simpleCommands() and auto-registers handlers](/envoy_series_redis_proxy_6.png)

## 3. Stats and docs — no changes needed

- Per-command stats are not a fixed list — they extract the command name directly from the request (`RedisCommandStats::getCommandFromRequest`), so no separate registration is needed.
- The `redis_proxy_filter.rst` documentation also does not enumerate the supported command list, so no documentation change is required.

## How to verify

After making the change, run the build below and verify the command:

```bash
bazel build //source/exe:envoy-static -c dbg
```

![Envoy Proxy Rebuild](/envoy_series_redis_proxy_7.png)


---

# Next post preview

The analysis is done. In the next post we will actually add `bitfield_ro` to `supported_commands.h`, get the tests passing, redeploy into the Minikube environment we built in week_5, and confirm that the `unknown command` error is gone.

fin.
