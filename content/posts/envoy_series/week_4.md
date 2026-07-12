---
title: "Envoy Proxy 기여 여정 #4: Envoy Proxy Build"
date: 2026-07-12T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"
series_order: 5
ShowToc: true
TocOpen: true
---
{{< series >}}

# 주제 및 배경
지난 시간에는 빌드를 위한 도커 이미지를 다운로드받고, 컨테이너를 켜는 것까지 진행했다. 이번 시간에는 그 컨테이너 안에서 Envoy Proxy를 빌드하고, Minikube 위에 Redis와 Envoy Proxy를 띄워 실제로 동작하는 모습을 확인해보자.

# 빌드
Envoy Proxy를 빌드해보자. 명령어는 놀랍도록 간단하다. Bazel을 통해 단 한 줄로 빌드할 수 있다.

내가 지난 [Envoy Proxy 개발환경 구성]({{< ref "posts/envoy_series/week_3(en).md#pulling-the-envoy-build-image" >}})에서 언급한 `envoy-build-ubuntu` Docker 이미지를 받아두어야만 가능하다. 아직 해당 이미지를 받지 않았다면, 해당 포스팅을 참고해서 컨테이너를 먼저 세팅해두자.

```shell
$ cd /source/source/
$ bazel build //source/exe:envoy-static -c dbg
```

빌드를 시작하면 아래 사진처럼 바로 진행된다. 참고로 대략적인 빌드 시간은 다음과 같다.

- **Mac M1 Max (RAM 32GB):** 약 3시간 30분
- **Mac M4 Pro (RAM 48GB):** 약 2시간
- **Mac M5 Pro (RAM 64GB, 현재 사용 중):** 약 4,619초

![Envoy Build](/envoy_series_build_envoy.png)

![Envoy Build Success](/envoy_series_build_envoy_success_time.png)

리소스가 충분하다면 위 명령어 한 번으로 빌드를 끝낼 수 있지만, 솔직히 Envoy Proxy는 빌드하기에 너무나도 큰 프로젝트다. 리소스가 부족한 경우 한 번에 메모리에 올려 빌드하는 것이 불가능하기 때문에, 리소스를 제한하여 빌드를 수행해야 한다. 이전 게시글에서 다음과 같이 강조했었는데, 기억하는가?

> 반드시 [Docker 설치 및 설정 가이드]({{< ref "posts/envoy_series/week_1(en).md#3-docker" >}})를 참고하여, `Docker Desktop을 설치하고 Resource 조정`을 하고 와야만 한다.

내 경험을 기준으로 정리하면, **RAM 30GB 이상은 있어야 안정적으로 빌드**할 수 있다. 다만 어디에도 명확한 기준이 적혀 있지는 않으니, 빌드 실패 메시지가 뜨면 한 번에 못 올리는 환경이라고 판단하면 된다. 이런 경우에는 CPU와 메모리를 제한하여 빌드를 돌리는 방법이 존재한다.

아래 예시는 호스트 리소스의 `0.4`만 사용하는 설정이다. 적게 설정할수록 빌드 시간이 길어지므로, 현재 단말 환경에 맞게 적당히 제한해서 사용하자. 내 경우에는 M1 Max 시절 Docker 리소스에 24GB 램을 할당하고, RAM과 CPU를 `0.8` 정도로 제한하여 빌드했다.

```shell
$ bazel build //source/exe:envoy-static -c dbg \
    --local_resources=cpu=HOST_CPUS*.4 \
    --local_resources=memory=HOST_RAM*.4
```

인고의 시간이 지나 빌드가 완료되면, Envoy Proxy 바이너리는 아래 경로에 생성된다.

```
/source/source/bazel-bin/source/exe/envoy-static
```

바이너리 사이즈를 보면 알겠지만, 굉장히 크다.

![Envoy Build Success](/envoy_series_build_envoy_success_binary.png)

생성된 바이너리를 호스트로 가져오자.

```shell
$ cd ~/Desktop/cncf/
$ docker cp b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2:/source/source/bazel-bin/source/exe/envoy-static .
```

# 인프라 구축
오늘은 빌드한 이미지를 그대로 띄우지는 않을 것이다. Redis와 Envoy를 별도로 띄울 예정이며, 다음 시간에 우리가 빌드한 버전의 Envoy로 교체해볼 것이다.

우선은 쿠버네티스 클러스터(Minikube)를 실행해준다.

```shell
$ minikube start
```

![Start Minikube](/envoy_series_minikube_start.png)

> 반드시 [Minikube 설치 가이드]({{< ref "posts/envoy_series/week_2(en).md#installing-minikube" >}})를 참고하여, `Minikube`와 `kubectl` 설치를 먼저 마치고 와야만 한다.

실행한 뒤, Minikube를 제어할 터미널은 반드시 별도로 켜서 아래 명령어를 수행해야 한다. 그 이유는 Envoy 빌드 환경은 클러스터 내의 컨테이너가 아니기 때문이다. 실제 서비스(Redis, Envoy)가 떠 있는 Minikube 환경과 격리되어 있으므로, Minikube에 뜨는 컨테이너와 구분을 하기 위함이다.

```shell
eval $(minikube docker-env)
```

준비가 끝났다면, [qerogram-example-envoy-redis](https://github.com/qerogram/example-envoy-redis) 레포의 `basic` 디렉터리 하위에 있는 yaml을 apply 해주면 된다.

```shell
git clone https://github.com/qerogram/example-envoy-redis
cd example-envoy-redis/basic

kubectl apply -f envoy-k8s.yaml
kubectl apply -f redis-k8s.yaml

kubectl get all
```

![Start Minikube](/envoy_series_minikube_deploy_basic.png)

Envoy Proxy의 경우 이미지를 한 번 다운로드 받아주면, Pod가 바로 실행된다.

```shell
docker pull envoyproxy/envoy:v1.38-latest
```

## Redis 접속 테스트
이제 접속 테스트를 해보자. 우선 redis-cli를 설치한다.

- **macOS:** `brew install redis`
- **Windows:** [Microsoft Archive Redis](https://github.com/microsoftarchive/redis/releases)에서 다운로드

Pod에 직접 붙기 전에 먼저 포트포워딩부터 진행한다. 위에서 `minikube docker-env`를 설정한 터미널을 켠 뒤, 아래와 같이 입력하면 Redis의 6379 포트로 접근할 수 있다.

```shell
kubectl port-forward deployments/redis 6379:6379
```

다른 터미널을 켜고 Redis에 연결한다.

```shell
$ redis-cli -p 6379
127.0.0.1:6379> hello
```

![Start Minikube](/envoy_series_minikube_test_redis.png)

## Envoy Proxy를 통한 Redis 접속 테스트
이어서 Envoy Proxy를 통해 Redis에 접근해보자. 16379 포트로 포트포워딩을 하면 된다.

```shell
kubectl port-forward deployments/envoy 16379:16379
```

다른 터미널을 켜고 Redis에 연결한다.

```shell
$ redis-cli -p 16379
127.0.0.1:16379> hello
```

![Start Minikube](/envoy_series_minikube_test_envoy.png)

# 마무리
이번 시간에는 Envoy Proxy를 빌드하고, 쿠버네티스 환경에서 Redis와 Envoy를 구성해 직접 접근하는 것까지 진행해봤다. 다음 시간에는 이번에 빌드한 Envoy 바이너리를 실제 서비스에 적용하는 과정으로 진행해보자.

fin