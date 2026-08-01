---
title: "[KR] Envoy Proxy 기여 여정 #5: Custom Envoy Proxy Deployment"
date: 2026-07-26T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 6
ShowToc: true
TocOpen: true
---
{{< series >}}

# 주제 및 배경

지난 회차([#4: Envoy Proxy Build]({{< ref "posts/envoy_series/week_4.md" >}}))에서는 Bazel로 Envoy Proxy를 빌드한 뒤, Minikube 위에 Envoy 공식 이미지(`envoyproxy/envoy:v1.38-latest`)와 Redis를 함께 띄워보았다. 그 회차의 목적은 쿠버네티스 인프라를 마련하는 것이었기 때문에, 실제로는 **내가 만든 Envoy가 동작하는지**까지는 검증하지 않았다.

이번 회차의 목표는 단순하다. 지난번에 빌드한 `envoy-static` 바이너리를 컨테이너 이미지로 묶어, week_4에서 띄운 배포를 **내가 빌드한 커스텀 Envoy로 교체**하는 것이다. 즉, "내가 만든 바이너리가 실제로 클러스터에서 살아 있는가"를 확인하는 단계다.

---

# 빌드

지금부터 진행할 작업은 단순하다. 호스트에 받아둔 `envoy-static` 바이너리를 [qerogram-example-envoy-redis](https://github.com/qerogram/example-envoy-redis) 레포의 `custom/` 디렉터리로 옮긴 뒤, 미리 준비된 매니페스트를 적용하기만 하면 된다.

> **선행 작업:** 반드시 [Envoy Proxy Build 가이드]({{< ref "posts/envoy_series/week_4.md#빌드" >}})를 참고하여, 빌드 산출물인 `envoy-static`을 호스트로 가져오는 작업을 먼저 마치고 와야 한다.

`custom/` 디렉터리 안에 있는 `Dockerfile`은 우리가 빌드한 Envoy 바이너리를 이미지에 심어 띄우는 구조다. 즉, **공식 Envoy 이미지에 우리가 만든 바이너리를 덮어씌우는 형태**라고 이해하면 된다.

> **Tip:** `docker build`로 생성된 이미지는 **호스트의 Docker daemon**에 저장된다. Minikube는 자체 Docker daemon을 사용하기 때문에, 빌드를 실행하기 전에 아래 명령어로 터미널의 Docker 컨텍스트를 Minikube로 맞춰두어야 한다. 이 부분을 빠뜨리면 `kubectl apply -f` 단계에서 `ImagePullBackOff` 오류가 발생한다.
>
> ```shell
> eval $(minikube docker-env)
> ```
>
> 자세한 배경은 [지난 회차의 인프라 구축 섹션]({{< ref "posts/envoy_series/week_4.md" >}})을 참고하자.

준비가 끝났다면, 아래의 명령어를 순서대로 실행하면 된다.

```shell
cd ~/Desktop/cncf/

# 지난 회차에서 빌드해둔 envoy-static이 현재 디렉터리에 있다고 가정한다.
git clone https://github.com/qerogram/example-envoy-redis

mv ./envoy-static example-envoy-redis/custom
cd example-envoy-redis/custom

# 커스텀 Envoy 이미지 빌드
docker build -t custom-envoy:v1 .

# Kubernetes 매니페스트 적용
kubectl apply -f envoy-k8s.yaml
kubectl apply -f redis-k8s.yaml

# 배포 상태 확인
kubectl get all
```

> **Note:** `envoy-k8s.yaml`은 `envoyproxy/envoy:v1.38-latest` 대신 우리가 빌드한 `custom-envoy:v1`을 사용하도록 이미 수정되어 있다. 따라서 별도로 이미지 이름을 바꿀 필요는 없다.

---

# 테스트

이제 실제로 동작하는지 확인해보자. 이전과 마찬가지로 `kubectl port-forward`를 사용해 호스트에서 직접 접근한다.

> **참고:** `port-forward` 명령어는 **포그라운드에서 계속 실행**되기 때문에, **서비스마다 별도의 터미널**을 띄워서 실행해야 한다. 한 터미널에서 두 명령을 동시에 실행하면 두 번째 명령이 블록되어 응답이 오지 않는다.

```shell
# 터미널 A
kubectl port-forward deployments/envoy 16379:16379

# 터미널 B
kubectl port-forward deployments/redis 6379:6379

# 터미널 C (검증용)
redis-cli -p 6379
redis-cli -p 16379
```

각 redis-cli 셸에 동일한 명령어인 `bitfield_ro`를 실행해 결과를 비교해보자. `bitfield_ro`는 읽기 전용 비트 연산 명령어로 비교적 최근에 추가된 명령이라, 일부 Envoy 프록시 구현에서는 아직 핸들러가 빠져 있을 수 있다.

```
# Redis에 직접 붙은 경우 (정상 응답)
127.0.0.1:6379> bitfield_ro
(error) ERR wrong number of arguments for 'bitfield_ro' command

# 우리가 빌드한 Envoy Proxy를 통해 붙은 경우 (명령어 자체가 미구현)
127.0.0.1:16379> bitfield_ro
(error) ERR unknown command 'bitfield_ro', with args beginning with:
```

![Envoy Command Check](/envoy_series_minikube_test_command_check.png)

핵심은 **두 응답의 종류가 다르다**는 점이다.

- **6379 포트 (Redis 직접):** "wrong number of arguments" — 명령어 자체는 구현되어 있고, 인자만 잘못 전달되었을 때의 정상 에러다.
- **16379 포트 (Envoy 경유):** "**unknown command**" — Envoy가 해당 Redis 명령어를 **인식조차 하지 못한다**는 의미다.

즉, 우리가 빌드한 Envoy의 Redis 프록시 필터에는 `bitfield_ro`를 처리하는 핸들러가 누락되어 있는 것이다. 이 발견이 본 시리즈의 다음 단계 출발점이 된다.

---

# 다음 회차 예고

본 시리즈를 따라오고 있다면 짐작했겠지만, 이제부터가 Envoy Proxy Contribution의 본게임이다. 미구현 명령어 `bitfield_ro`를 Redis 프록시 필터에 실제로 구현해보는 작업을 진행한다.

참고로, 코드를 읽을 줄 안다면 사실 내가 여기서 더 알려주지 않아도, **LLM**(ChatGPT, Claude, Grok, Gemini 등) 중 무엇이든 써서 코드를 수정한 뒤 빌드하고, Kubernetes로 쉽게 배포하여 명령어를 테스트하면 된다.

다만 그렇게 무책임하게 끝내는 것은 아쉬우니, 이번 시리즈에서는 실제로 Hands-On을 거치며 Redis Proxy에 명령어를 간단하게 추가해보는 방법을 소개하며 일부 코드베이스를 함께 들여다보겠다.

fin.
