---
title: "Envoy Proxy 기여 여정 #2: 문제 발견과 이슈 분석"
date: 2026-06-14T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy"]
series: "contribution_envoy"          # ← 여기서 배열(-)로 바꾸기!]
series_order: 3
ShowToc: true           # ← 목차 활성화
TocOpen: true       # ← 기본으로 펼쳐서 보여줌 (false로 하면 접힘)
---
{{< series >}}

# 주제 및 배경
본 시리즈를 통해 제가 하고 싶은 것은, Envoy Proxy를 통해 Redis와 연결시 사용할 수 있는 기능을 구현하는 방법에 대해서 다루고자 합니다.

보안을 위한 과제를 수행하다가, Envoy Proxy에 기여할 기회가 있었습니다.

이 과정에서 CNCF Ambassador에 대해 알게 되었고, Ambassador가 되기 위한 기여를 진행하면서 블로그 포스팅을 준비하게되었습니다. 많은 분들이 Envoy Proxy에 대해 기여할 수 있어서 더 멋진 프로그램이 되길 바랍니다. 저도 이 과정을 통해 CNCF Ambassador가 될 수 있으면 좋겠습니다 😆


# Envoy Proxy란?
CNCF의 3번째 Graduated Project인 Envoy Proxy는 2026년 5월 5일 기준 star가 27,912이나 되는 대형 프록시 프로젝트입니다.
쿠버네티스에서 주로 사용되는 프록시인 만큼 성능적으로는 충분히 성숙도가 높은 프로젝트입니다.


# 하고자 했던 과제
* Envoy 프록시가 필요했던 이유는, DB 서버 접근 제어를 목표로 하고 있습니다.
* 기존에는 직접 접근이 가능했던 DB를, teleport를 통해 서버 접근을 가능하도록 설계를 바꾸려고 했습니다.
	* eks + istio 환경
* 이 과정에서 teleport에서 AWS Elasticache 를 연결하기 위해서는 '전송 중 암호화(TLS)'를 켜야만 하고, 해당 기능을 키면 성능이 다소 떨어지게 됩니다.
* 즉, 이 과정에서 TLS를 위해 envoy proxy를 두어서 해결하였습니다.
* 근데 이렇게 되니... 지원하지 않는 기능들이 존재해서, GUI 도구를 사용할 수 없었습니다.
* 그래서 Envoy Proxy에 지원하지 않는 기능을 추가하자!라는 생각으로 진행을 하였고, 실제 기여에 성공했습니다.


## 본 시리즈에서 다뤄볼 내용
* 환경 세팅
* 빌드 방법
* 연결 방법
* 디버깅
* PR, ... , Approve
* 적용 후 릴리즈까지..


