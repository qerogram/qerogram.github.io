---
title: "Contributing to Envoy Proxy: A Journey with Redis Integration"
date: 2026-05-03T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 1
ShowToc: true
TocOpen: true
---
{{< series >}}

# Overview and Background

This series documents **how to implement and contribute Redis proxy features to Envoy Proxy**, covering the entire journey from development environment setup to merging a contribution into the Envoy project. Rather than theoretical knowledge, this series shares practical experience from a real-world production scenario.

The motivation for this series came from addressing a Redis connectivity challenge during a security-focused infrastructure task. Through solving this problem using Envoy Proxy, I discovered the CNCF Ambassador program and was inspired to contribute missing features directly to the Envoy Proxy project. Ultimately, the contribution was accepted and merged into the main branch.

My hope is that this series will inspire more engineers to contribute to Envoy Proxy and advance the CNCF ecosystem. And who knows—perhaps this contribution journey will help me become a CNCF Ambassador too! 🚀


# What is Envoy Proxy?

**Envoy Proxy** is the 3rd Graduated Project in the CNCF, having accumulated 27,912 GitHub stars as of May 2026. It is a large-scale, production-grade proxy platform widely used as a core component in Kubernetes-based Service Mesh implementations. Known for its high performance and stability, Envoy has become the de facto standard for modern cloud-native networking.

**Key Capabilities:**
- Layer 4 and Layer 7 traffic routing and control in microservices environments
- TLS/SSL termination and certificate management
- Protocol-specific proxying for Redis, MySQL, PostgreSQL, and other databases
- Real-time observability, metrics, and structured logging
- Dynamic configuration and gradual rollout support


# The Technical Challenge We Faced

## Background: Strengthening Security with Architecture Redesign

The original system allowed applications to connect directly to AWS ElastiCache (Redis) without intermediaries. To enhance security posture, we decided to adopt a **Zero Trust architecture using Teleport** (running on EKS + Istio environment).

## The Problem

To securely connect through Teleport to AWS ElastiCache, **TLS (encryption in transit) must be enabled**. However, this introduced several issues:
- ❌ Enabling TLS degrades Redis performance by approximately
- ❌ Teleport's GUI-based client tools lack support for TLS-encrypted connections
- ❌ Compatibility issues with existing monitoring and management tools

## The Solution: Introducing Envoy Proxy

We deployed **Envoy Proxy as a Redis proxy layer** to address these challenges:
- Envoy handles TLS termination → applications communicate in plaintext
- Achieves both performance optimization and security hardening
- Enables additional monitoring and traffic control capabilities

## A New Problem: Feature Gap

At that time, Envoy's Redis proxy implementation was missing **RESP3 protocol support**. This limitation meant:
- Inability to leverage newer Redis 7.0+ features
- GUI-based Redis client tools could not function properly
- Debugging and development workflows became constrained

## Moving Toward Contribution

Faced with this limitation, we asked: **"What if we implement the missing feature ourselves?"** This question led us to contribute directly to the Envoy Proxy project. We successfully implemented the required functionality, submitted a PR, and achieved approval from the core maintainers.


## What This Series Covers

This series walks through each step of the Envoy contribution experience:

1. **Setting Up the Development Environment** — Installing dependencies, configuring the Envoy build system, and preparing your local workspace
2. **Code Analysis and Design** — Understanding the existing Redis proxy implementation, identifying the gap, and planning your contribution
3. **Implementation** — Writing code to add RESP3 protocol support, including unit and integration tests
4. **Local Testing** — Validating the implementation against real ElastiCache instances and measuring performance impact
5. **Submitting and Responding to Code Review** — Creating a pull request on GitHub, addressing reviewer feedback, and iterating on suggestions
6. **Community Interaction** — Collaborating with Envoy maintainers, understanding the review process, and achieving approval
7. **Release and Deployment** — Merging into the main branch, being included in an official release, and deploying to production

Throughout each step, we'll share the problems encountered, solutions implemented, and lessons learned.

