---
title: "Envoy Proxy 기여 여정 #1: 개발 환경 구축"
date: 2026-05-31T22:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 2
ShowToc: true
TocOpen: true
---
{{< series >}}

# 들어가며
본 시리즈를 시작하며, Envoy Proxy가 얼마나 대단한 지는 다들 아실거라고 생각합니다.
Istio환경에서도 실질적으로 표준에 준하는 Proxy이자, 굉장히 성숙도가 높은 Proxy입니다.
CNCF의 3번째 `Graduated Project`입니다.


# Envoy는 무엇으로 구성되고, 어떻게 생겼는가 ?
- 가장 중요한 것은 Proxy라는 점.
- 빌드 도구 Bazel로 빌드를 진행함.
- 작성된 언어는 C++로 되어 있음.
	> [!WARNING] 참고
	> 최근에 High Level Language로 코딩을 시작하신 분들은 이 과정이 쉽지 않을 것 같습니다. <br/>
	> 여담이지만, 저는 C++는 조금 할 줄 아는 상태이며, <br/>
	> Reverse Enginnering으로 C와 그렇게 낯설지는 않은 사이였습니다. <br/>
	> 그런 저도 레퍼런스가 없어서 굉장히 많은 삽질을 하면서 익혔습니다. <br/>
	> Envoy Proxy에 기여하려는 분 혹은 AI Agent한테도 `제가 작성하는 그 내용이 큰 도움이 될 거라 기대`합니다.



# 준비해야할 것
우리는 `Container`에서 빌드를 하고 최종적으로는 `k8s(kubernetes)에 배포`하는 것을 목표로 하고 있습니다. 
구체적으로는 모두가 할 수 있게 돈이 들지 않도록 minikube라고 하는 로컬 환경에 쿠버네티스 클러스터를 구축하고, 가는 방식으로 진행하려고 합니다.

1. 제일 먼저 필요한 것은 github입니다. github desktop 혹은 github ci를 먼저 설치합니다.
2. 소스 코드를 작업해볼 것은 VSCode 제품을 사용하여, 코드를 보고, 추가를 해볼 것입니다.
3. 다음으로 필요한 것은 Docker 입니다. 우리는 최종적으로 Docker에서 빌드할 것입니다.
4. 마지막으로 환경이 동작할 minikube 구축, 미니큐브는 다음 편에 구축할 예정입니다.


# Blue Print

내가 경험했던 프로젝트는 다음과 같다.
![Blueprint Overall](/envoy_series_blueprint_overall.png)

다만 이 경우에서 사실 본질적으로 중요한 것은 Envoy Proxy -> Redis 간의 기능이 완벽하지 않았기에, 내가 Envoy Proxy에 기여하는 경험을 하게 됐다.

본 시리즈에서는 아래와 같이, Envoy-Proxy -> Redis에 대한 내용을 다루고자 한다.
![Blueprint](/envoy_series_blueprint_target.png)

즉, Redis Filter에 대한 내용을 주로 다룰 예정이다.


## 1. Github 설정
1. 우선 Github 부터 가입
- https://github.com/ 에 들어가서 Sign Up을 하면 된다.

<br/>

2. github 소프트웨어를 설치
아래의 셋 중 하나를 설치해준다 .
- [github Dekstop](https://desktop.github.com/download/)도 좋고 
- [git-scm](https://git-scm.com/)를 설치해도 좋다. 
- [github cli](https://cli.github.com/)를 설치해도 좋다. 

<br/>

3. 계정의 권한을 이용하기 위해, 토큰 설정을 해준다.
참조할만한 링크는 한국어로 된 링크지만, https://www.postype.com/@cpuu/post/10570269 사이트를 참조해서 설정해주면 된다.
- 깃허브에는 "본인 계정 권한을 사용할 수 있는 컴퓨터 정보를 등록"할 수 있다.
- https://github.com/settings/profile 페이지로 이동한 뒤, SSH and GPG Keys에 등록을 해주면 된다.
- 명령어를 실행하면 만들 수 있다. (맥, 우분투에서는 terminal, 윈도우에서는 Powershell로 명령어를 수행하면 된다.) 
	```shell
	ssh-keygen -t ed25519 -C "qerogram@gmail.com"
	```

<br/>

-  ~/.ssh/id_ed25519.pub 파일에 담긴, ssh-ed25519로 시작하는 문자열을 github 홈페이지의 SSH and GPG Keys에 등록해주면 된다.
	- 잘 되지 않는다면 권한 설정을 해주면 된다.
	```shell
	chmod 400 ~/.ssh/id_ed25519.pub
	```

<br/>

4. 설치를 잘했다면 [Envoy Proxy](https://github.com/envoyproxy/envoy)에 들어가서 설치하자.
- 레포의 code 버튼을 클릭하고, ssh를 선택 한 뒤 경로를 가져오자.
![SSH](/envoy_series_git_clone.png)

아래와 같이 git clone 경로에 넣어서 명령어를 만들어주고, 실행하면 된다.
```shell
git clone git@github.com:envoyproxy/envoy.git
```



## 2. VSCode
1. 다음의 링크에서 [VS Code](https://code.visualstudio.com/download)를 설치해준다.

<br/>

2. VS Code의 Extension에서 Docker, Container 등을 검색해서 설치해준다.
![Docker Settings](/envoy_series_vscode_extension.png)


<br/>


## 3. Docker
1. 수많은 컨테이너 소프트웨어가 있지만, 나는 Docker를 꽤 선호한다.
[Docker Desktop](https://www.docker.com/products/docker-desktop/)을 설치해준다. docker-ce를 받던 뭐든 상관없지만, Desktop이 사용이 편리하다.

<br/>

2. `Docker Desktop에서 리소스를 먼저 조정`해준다. 리소스가 작다면, 빌드에 아주 오랜시간이 소요되곤 한다. M5 Pro 맥북을 기준으로, 메모리 48기가를 세팅해주면 리소스 조정 없이 사용이 가능하고, 디스크는 200GiB 이상 요구된다.
![Docker Settings](/envoy_series_docker_settings.png)

<br/>


## 4. 마치며,
여기까지 했으면, 필요한 소프트웨어는 모두 설치 완료했다.


Fin.
