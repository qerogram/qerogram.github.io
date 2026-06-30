---
title: "Envoy Proxy 기여 여정 #3: 빌드 환경 구축 - 2"
date: 2026-06-28T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"
series_order: 4
ShowToc: true
TocOpen: true
---
{{< series >}}

# 주제 및 배경
지난 포스팅에서는 우리가 최종적으로 실행할 환경인 쿠버네티스(Minikube)를 구축했다. 이번 포스팅에서는 Envoy Proxy를 빌드할 수 있는 환경을 구축해보자.

이 부분에 대한 설명이 공식 문서에서도 굉장히 모호해서, 내 경우에는 엄청난 삽질을 하다가 그제서야 가장 쉬운 빌드 방법을 찾아낼 수 있었다. 이번 포스팅은 그 과정을 공유하기 위해 작성한다.

---

# Docker Build Image 설치

> **참고**
> 이 문단을 읽기 전에 반드시 [Docker 설치 및 설정 가이드]({{< ref "posts/envoy_series/week_1(en).md#3-docker" >}})를 참고해 `Docker Desktop 설치`와 `Resource 조정`을 먼저 마치고 와야만 한다.

Envoy Proxy 빌드는 Docker Container 위에서 수행할 것이다. 빌드용 Container Image는 Docker Hub에 이미 올라와 있다. Envoy Proxy의 빌드를 위해 Ubuntu 등 별도의 Linux 환경을 직접 구축하기보다는, 공식적으로 제공하는 이미지를 받아 사용하는 것이 훨씬 수월하다. 나 역시 이 방법을 모른 채 직접 빌드하고 명령어를 하나씩 찾아가느라 너무 고생했다. 같은 시행착오를 반복하지 않도록, 이번에는 가장 쉬운 가이드로 바로 안내해본다.

아래의 [Docker Hub Registry](https://hub.docker.com/r/envoyproxy/envoy-build-ubuntu/tags)로 들어가서, 가장 최신 Tag 중 Prefix가 `full-`로 시작하는 Image를 선택하면 된다.

![Docker Hub Image](/envoy_series_docker_hub_image.png)

본문 기준으로는 다음 이미지를 사용한다.

```
envoyproxy/envoy-build-ubuntu:full-86873047235e9b8232df989a5999b9bebf9db69c
```

명령어 그대로 pull을 진행하자.

```shell
docker pull envoyproxy/envoy-build-ubuntu:full-86873047235e9b8232df989a5999b9bebf9db69c
```

pull이 완료되면 Docker Desktop의 `Images` 탭에서 아래와 같이 확인할 수 있다.

> **참고:** 해당 이미지의 용량은 약 12GB 수준으로 굉장히 크다.

![Docker Image](/envoy_series_docker_image_tab.png)

---

# Build 환경 실행
먼저 Github의 Envoy Proxy 레포지토리를 Clone 한다. 이 시리즈를 매주 따라오고 있다면, 이미 로컬에는 클론이 있을 테지만 아마 최신 버전은 아닐 것이다. Clone한 디렉터리에서 `git pull` 명령어를 입력해 최신 상태로 맞춰주자.

> **참고:** Clone을 아직 하지 않았다면, [Github 설치 및 Clone 가이드]({{< ref "posts/envoy_series/week_1(en).md#1-github-configuration" >}})를 참고해 `GitHub 설치`와 `Envoy Proxy 레포지토리 Clone`을 먼저 진행한 뒤 다시 와야만 한다.

이제 Envoy Proxy 디렉터리의 상위 폴더로 이동해서 폴더명을 변경하고, Docker Container를 실행한다.

```shell
$ mv envoyproxy source/
$ docker run -itd -v $(pwd):/source -w /source  envoyproxy/envoy-build-ubuntu:full-86873047235e9b8232df989a5999b9bebf9db69c
```

실행 후 출력되는 문자열이 컨테이너의 `Container ID`다.

```
b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2
```

Docker에서는 이 `Container ID`를 기준으로 파일 이동(복사), 상태 변경(시작, 일시중지, 삭제) 등을 수행한다. `docker exec` 명령으로 Container 내부의 Shell을 실행하면, Host에서 Container로 연결해 작업할 수 있다.

```shell
~/Desktop/cncf                                                         11:43:00
❯ docker run -itd -v $(pwd):/source -w /source  envoyproxy/envoy-build-ubuntu:full-86873047235e9b8232df989a5999b9bebf9db69c
b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2

~/Desktop/cncf                                                                                                                    11:43:08
❯ docker exec -it b8edaacc488a91072db4559673819dad785795ba3dd9dc5f47ee5c848454abd2 /bin/bash
root@b8edaacc488a:/source# id
uid=0(root) gid=0(root) groups=0(root)
```

위 출력에서 주목할 부분은 다음과 같다.

- 프롬프트가 호스트의 경로에서 `root@b8edaacc…:/source#`로 바뀐 것을 확인할 수 있다. 이는 컨테이너 내부에 들어왔다는 뜻이다.
- `id` 명령 결과가 `uid=0(root)`인데, 빌드 이미지는 기본적으로 root 권한으로 동작한다. 마운트된 볼륨에 파일을 생성할 때 권한 문제로 인한 삽질을 피할 수 있다.

![Docker Container](/envoy_series_docker_run_container.png)

다음 포스팅에서는 이어서 빌드 명령어를 실행해 실제로 Envoy Proxy를 빌드해보자.

Fin.