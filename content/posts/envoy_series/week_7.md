---
title: "[KR] Envoy Proxy 기여 여정 #7: bitfield_ro 빌드 및 검증"
date: 2026-08-21T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"
series_order: 8
ShowToc: true
TocOpen: true
---
{{< series >}}

# 주제 및 배경

지난 회차에서는 `bitfield_ro`를 `simpleCommands()`에 추가해야 한다는 결론을 내렸다. 이번 회차에서는 변경 사항이 반영된 Envoy 바이너리를 week_5에서 구축한 Minikube 클러스터에 재배포하고, 새 명령어가 전체 경로에서 정상적으로 동작하는지 확인해보자.

# 사전 작업

week_5에서 했던 것처럼, 호스트에 받아둔 `envoy-static` 바이너리를 [qerogram-example-envoy-redis](https://github.com/qerogram/example-envoy-redis) 리포지터리의 `custom/` 디렉터리로 옮긴 뒤, 준비된 매니페스트를 적용하면 된다.

먼저 컨테이너에서 빌드된 바이너리를 호스트로 가져온다.

> 기억나지 않는다면 [Envoy Proxy Build 가이드]({{< ref "posts/envoy_series/week_4.md#빌드" >}})를 참고해 빌드 산출물인 `envoy-static`을 호스트로 가져오는 방법부터 다시 확인하자.
>
> ```shell
> $ cd ~/Desktop/cncf/
> $ docker cp b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2:/source/source/bazel-bin/source/exe/envoy-static .
> ```

`custom/` 디렉터리 안에 있는 `Dockerfile`은 우리가 빌드한 Envoy 바이너리를 이미지에 포함해 컨테이너로 실행하는 역할을 한다. 즉, **공식 Envoy 이미지에 우리가 만든 바이너리를 덮어씌우는 형태**라고 이해하면 된다.

> **Tip:** `docker build`로 생성된 이미지는 **호스트의 Docker daemon**에 저장된다. Minikube는 자체 Docker daemon을 사용하기 때문에, 빌드 전에 아래 명령어로 터미널의 Docker 컨텍스트를 Minikube로 전환해야 한다. 이 과정을 빠뜨리면 `kubectl apply -f` 단계에서 `ImagePullBackOff` 오류가 발생한다.
>
> ```shell
> eval $(minikube docker-env)
> ```
>
> 자세한 배경은 [지난 회차의 인프라 구축 섹션]({{< ref "posts/envoy_series/week_4.md" >}})을 참고하자.

준비가 끝났다면 아래 명령어를 순서대로 실행하자. 먼저 기존 Envoy 리소스를 정리하고, 이전에 만든 Docker 이미지를 삭제한다.

```shell
cd ~/Desktop/cncf/
kubectl delete -f envoy-k8s.yaml
docker rmi custom-envoy:v1
```

# 이미지 재배포

정리가 끝났다면 컨테이너 이미지를 다시 빌드한 뒤 Kubernetes에 재배포한다.

```shell
# 지난 회차에서 빌드한 envoy-static 바이너리가 현재 디렉터리에 있다고 가정한다.
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

Pod가 모두 실행되면 터미널을 두 개 열어 다시 포트 포워딩을 실행하자.

```shell
kubectl port-forward deployments/redis 6379:6379
kubectl port-forward deployments/envoy 16379:16379
```

아래는 새 바이너리를 재배포하기 전에 Envoy를 통해 Redis에 `bitfield_ro` 명령어를 실행한 결과다.

![업데이트된 Envoy 바이너리 재배포 전](/envoy_series_redis_proxy_week7_1.png)

```shell
bitfield_ro qero get u8 0
```

변경 사항이 반영된 바이너리를 배포한 뒤에는 아래와 같이 정상적으로 동작한다.

![업데이트된 Envoy 바이너리 재배포 후](/envoy_series_redis_proxy_week7_2.png)

# 커버리지 테스트

`bitfield_ro`가 이제 정상적으로 인식되고 전달되는 것을 확인할 수 있다. 이번 변경은 `simpleCommands()`에 명령어를 추가하는 작업이므로, 기존 command splitter 테스트가 이 등록 경로를 이미 검증한다. 따라서 별도의 테스트를 추가할 필요는 없다.

포스팅을 마무리하기 전에 이번 변경 사항이 프로젝트의 커버리지 기준을 충족하는지도 확인해보자.

Envoy Proxy를 빌드했던 컨테이너에 다시 접속한 뒤 아래 명령어를 실행한다. 현재 테스트가 소스 코드의 어느 정도를 실행하는지 확인할 수 있다.

> 참고로 커버리지 테스트를 실행하면 Bazel 캐시가 무효화될 수 있어 이후 빌드 시간이 다소 늘어날 수 있다.

```shell
cd /source/source
./ci/do_ci.sh coverage //test/extensions/filters/network/redis_proxy/...
```

![Coverage Test 1](/envoy_series_redis_proxy_week7_3.png)

아래와 같은 결과를 확인할 수 있다. Redis Proxy의 경우 메인테이너들이 정한 기준에 따라 커버리지가 **96.6%를 넘어야 한다**. 이 기준에 미치지 못하면 Pull Request의 CI 검사가 실패하므로, 병합하기 전에 테스트를 추가하거나 필요한 변경을 통해 커버리지를 높여야 한다.

![Coverage Test 2](/envoy_series_redis_proxy_week7_4.png)

97.2%를 기록했으므로 커버리지 기준을 충족했다.

커버리지 기준은 `test/coverage.yaml`에 정의되어 있다. 필터에 별도의 기준이 없으면 기본 목표는 **96.6%**다.

![Coverage 확인](/envoy_series_redis_proxy_week7_5.png)

여기까지 따라 했다면 커스텀 Envoy 이미지를 빌드해 사내 또는 개인용 프라이빗 컨테이너 레지스트리에 올려 사용할 수 있다.

# 다음 회차 예고

다음 회차부터는 Pull Request를 제출하는 절차를 살펴보자. 변경 사항이 upstream에 병합되면 더 이상 별도의 프라이빗 컨테이너 레지스트리에 올릴 필요 없이 공식 Envoy 릴리스를 사용하면 된다.

fin.
