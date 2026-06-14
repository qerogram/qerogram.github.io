---
title: "[KR] Envoy Proxy 기여 여정 #2: 빌드 환경 구축 - 1"
date: 2026-06-14T12:00:00+09:00
# date: 2026-06-14T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"          # ← 여기서 배열(-)로 바꾸기!]
series_order: 3
ShowToc: true           # ← 목차 활성화
TocOpen: true       # ← 기본으로 펼쳐서 보여줌 (false로 하면 접힘)
---
{{< series >}}

# 주제 및 배경
쿠버네티스 서버 설치에 대해서 다루려고 한다. 
일반적인 쿠버네티스 서비스를 이용하기에는 많은 비용이 들기 때문에, 우리는 비용 문제로 인해 로컬에서 테스트해볼 수 있는 단일 노드(PC)의 쿠버네티스를 구축하고자 한다.

구축한 뒤, 주로 이용하는 Utilization 도구인 kubectl, k9s 도구도 설치를 진행해본다.


# Minikube 설치
우선은 미니큐브를 설치해주자.
*  [minikube sig](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download) 사이트에 나오는 설치 방법을 그대로 따라가면 된다.
* 참고 : https://kubernetes.io/ko/docs/tutorials/hello-minikube/

나는 실리콘 맥을 사용 중이어서, 편하게 Homebrew로 설정하기 위해 다음과 같이 누르면 설치 방법이 아래와 같이 나오게 된다 .
![Minikube Sig Image](/envoy_series_minikube_install.png)

즉, 아래의 명령어를 입력하면 설치가 진행된다.

```shell
brew install minikube
```
# 쿠버네티스 제어 준비하기

직후에는, 터미널에서 아래와 같은 명령어를 입력하면 Kubernetes Cluster가 생성된다.

```shell
minikube start
```

아래와 같이 minikube 가 실행되기 위해서는 docker가 반드시 켜져있어야한다.
만약 도커를 사용 불가능해서 rancher, podman 등 다른 컨테이너 가상화 도구를 사용한다면 다음과 같이 driver 옵션을 주어 선택이 가능하다.
* 사용 가능한 driver 옵션은 [Minikube Driver Option](https://minikube.sigs.k8s.io/docs/drivers/)에서 확인이 가능하다.
```shell
minikube start --driver=podman
```
![Minikube Start](/envoy_series_minikube_start.png)


쿠버네티스를 제어하기 위해서 기본적으로 사용하는 Utilization Tool은 kubectl이 있다.

* kubectl 설치에 대한 가이드는 다음의 사이트에 있다. Linux, MacOS, Windows 시스템에 대해서 지원하고 있다.
	* [Kubernetes.io Install Guide](https://kubernetes.io/ko/docs/tasks/tools/)

<br/>

* kubectl의 사용법은 다음 사이트를 참고하자
	* [Kubernetes.io Use Guide](https://kubernetes.io/ko/docs/reference/kubectl/)


설치가 완료되었다면, `kubectl get all` 명령어를 통해서 연동이 되었는지 확인해주면 된다.
* 현재 내 경우에는, redis, envoy 에 대한 실험이 끝나서 관련 리소스가 노출되고 있다. 독자분들은 service/kubernetes 가 잘 나오는 지만 확인해주시면 된다.
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
replicaset.apps/envoy-576d5d575    1         1         1       13d
replicaset.apps/redis-748cffd85d   1         1         1       13d
```



Fin.
