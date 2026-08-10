---
title: "[KR] Envoy Proxy 기여 여정 #6: 이슈 분석 - bitfield_ro는 어디에 추가하는가"
date: 2026-08-09T12:00:00+09:00
draft: false
tags: ["cncf", "envoy", "redis-proxy", "open-source"]
series: "contribution_envoy"
series_order: 7
ShowToc: true
TocOpen: true
---
{{< series >}}

# 주제 및 배경

지난 회차([#5: Custom Envoy Proxy Deployment]({{< ref "posts/envoy_series/week_5.md" >}}))에서는 직접 빌드한 Envoy를 Minikube 클러스터에 배포하고, Redis에 직접 붙었을 때와 Envoy를 경유했을 때의 응답 차이를 비교해 `bitfield_ro` 명령어가 프록시에서 **unknown command**로 떨어지는 것을 확인했다.

이번 회차에서는 바로 그 문제를 고치기 위해, Envoy Redis Proxy 코드베이스에서 **어디를 어떻게 건드려야 하는지** 분석해보자. 아직 코드를 수정하지는 않고, 수정 지점을 정확히 특정하는 것이 목표다.

---

# BITFIELD_RO는 어떤 명령어인가

`BITFIELD_RO`는 Redis 6.0에서 추가된 `BITFIELD`의 읽기 전용 버전이다. `BITFIELD`가 `GET`/`SET`/`INCRBY` 서브커맨드로 하나의 키에 담긴 여러 비트필드를 읽고 쓸 수 있다면, `BITFIELD_RO`는 그중 `GET`만 허용하는 변형이다.

라우팅 관점에서 중요한 점은 두 명령 모두 **첫 번째 인자인 키 하나만 해싱하면 된다는 것**이다. 즉, Envoy Redis Proxy가 이미 가진 "단일 키 → 단일 샤드" 라우팅 로직으로 충분히 처리할 수 있고, 응답을 병합하는 새로운 핸들러가 필요하지 않다. 이 판단이 이번 분석의 핵심이다.

---

# Redis Proxy의 명령 구성

## 모듈 위치

먼저 코드베이스에서 Redis Proxy 관련 코드의 위치를 정리하면 아래와 같다.

| 경로 | 역할 |
|---|---|
| `source/extensions/filters/network/redis_proxy/` | Redis proxy 네트워크 필터 본체 (프록시, 커맨드 스플리터, 라우터, 커넥션 풀) |
| `source/extensions/filters/network/common/redis/` | RESP 코덱, Redis 클라이언트, 명령어 분류 등 공용 프리미티브 |
| `source/extensions/clusters/redis/` | Redis 전용 클러스터 타입 (토폴로지 인식 LB) |
| `api/envoy/extensions/filters/network/redis_proxy/v3/redis_proxy.proto` | 필터 설정 스키마 |
| `test/extensions/filters/network/redis_proxy/` | 단위/통합 테스트 |

## 요청이 처리되는 흐름

클라이언트가 보낸 `bitfield_ro` 요청이 어떤 경로를 지나는지 따라가 보자.

1. `proxy_filter.cc` — `ProxyFilter::onData()`가 수신 바이트를 RESP 디코더(`decoder_->decode`)에 넘긴다
2. 디코딩이 완료되면 `ProxyFilter::onRespValue()`가 호출되고, 여기서 `splitter_.makeRequest()`로 요청을 넘긴다
3. `command_splitter_impl.cc` — `InstanceImpl::makeRequest()`가 명령어 이름으로 **룩업 테이블**에서 핸들러를 찾는다
![supported_commands.h — simpleCommands() 목록. "bitfield" 옆이 bitfield_ro 추가 위치다](/envoy_series_redis_proxy_1.png)

4. 핸들러가 키를 해싱해 라우터(`router_impl.cc`)에서 upstream 풀을 고르고, 커넥션 풀(`conn_pool_impl.cc`)로 요청을 보낸다

여기서 3번의 룩업 테이블에 등록되어 있지 않은 명령어는 곧바로 에러 응답을 받는다. 우리가 본 `unknown command 'bitfield_ro'` 에러가 바로 여기서 나온 것이다.

## 명령어가 등록되는 구조

그렇다면 룩업 테이블에는 누가 등록하는가? 명령어 지원 여부는 `common/redis/supported_commands.h`의 `SupportedCommands`가 정의하고, 스플리터가 이 목록을 읽어 등록한다. 명령어는 라우팅 방식에 따라 아래처럼 셋(set)으로 나뉜다.
![command_splitter_impl.cc — simpleCommands()를 순회하며 핸들러를 자동 등록하는 루프](/envoy_series_redis_proxy_2.png)

![command_splitter_impl.cc — simpleCommands()를 순회하며 핸들러를 자동 등록하는 루프](/envoy_series_redis_proxy_3.png)

- `simpleCommands()` — 단일 키로 해싱해서 한 샤드로 보내는 명령 (`get`, `set`, `bitfield` 등)
- `multiKeyCommands()` — `del`, `mget`, `mset` 등 다중 키 명령
- `evalCommands()` — `eval`, `evalsha`
- `hashMultipleSumResultCommands()` — 여러 샤드에 보내고 응답을 합산하는 명령 (`exists` 등)
- `ClusterScopeCommands()` — 모든 샤드에 보내는 명령 (`flushall`, `config` 등)
- `randomShardCommands()` — 무작위 샤드로 보내는 명령 (`randomkey` 등)
- `transactionCommands()` — `multi`, `exec` 등 트랜잭션 명령
- `writeCommands()` — 쓰기 명령 셋. `isReadCommand()`는 이 셋에 **없으면** 읽기 명령으로 판단한다

그리고 커맨드 스플리터는 시작할 때 **이 셋들을 순회하면서 핸들러를 자동 등록**한다.

```cpp
// source/extensions/filters/network/redis_proxy/command_splitter_impl.cc
for (const std::string& command : Common::Redis::SupportedCommands::simpleCommands()) {
  addHandler(scope, stat_prefix, command, latency_in_micros, simple_command_handler_);
}
```

![command_splitter_impl.cc — simpleCommands()를 순회하며 핸들러를 자동 등록하는 루프](/envoy_series_redis_proxy_4.png)

> **Note:** 결국 새 명령어 추가는 "이 명령이 어떤 셋에 속하는가"를 판단하는 문제로 귀결된다. 셋에 넣기만 하면 룩업 테이블 등록은 자동으로 따라오기 때문이다.

---

# bitfield_ro를 추가하려면 건드려야 할 곳

앞서 분석한 대로 `BITFIELD_RO`는 단일 키 해싱이면 충분하므로 `simpleCommands()`가 정답이다. 이제 파일별로 정리해보자.

## 1. supported_commands.h — 수정의 본체

```diff
 // source/extensions/filters/network/common/redis/supported_commands.h
-        "bitcount", "bitfield", "bitpos", "decr", ...
+        "bitcount", "bitfield", "bitfield_ro", "bitpos", "decr", ...
```

![command_splitter_impl.cc — simpleCommands()를 순회하며 핸들러를 자동 등록하는 루프](/envoy_series_redis_proxy_5.png)


> **Tip:** 읽기 전용 명령이므로 `writeCommands()`에는 추가하면 **안 된다**. `isReadCommand()`는 `!writeCommands().contains(command)`로 판정하므로, 넣지 않는 것만으로 읽기 명령으로 처리된다. 이 판정은 이후 읽기 전용 라우팅(레플리카로 보내는 정책 등)에 영향을 준다.

## 2. command_splitter_impl.cc — 수정 불필요

`InstanceImpl` 생성자가 `simpleCommands()` 전체를 순회하며 `simple_command_handler_`에 등록하므로, 셋에 추가만 하면 룩업 테이블에 자동 등록된다. 새 핸들러가 필요한 것은 `MGET`/`SCAN`처럼 응답을 분할·병합하는 특별한 동작이 필요한 명령뿐이다.
![command_splitter_impl.cc — simpleCommands()를 순회하며 핸들러를 자동 등록하는 루프](/envoy_series_redis_proxy_6.png)

## 3. 통계/문서 — 수정 불필요

- 명령별 통계는 고정 목록이 아니라 요청에서 명령어를 그대로 추출하는 방식(`RedisCommandStats::getCommandFromRequest`)이라 별도 등록이 필요 없다.
- `redis_proxy_filter.rst` 문서도 지원 명령 목록을 나열하지 않아 변경이 필요 없다.

## 검증 방법

수정 후 아래 빌드를 해보자, 그 후에 명령어를 실행해보면 된다.

```bash
bazel build //source/exe:envoy-static -c dbg
```

![Envoy Proxy Rebuild](/envoy_series_redis_proxy_7.png)


---

# 다음 회차 예고

분석은 끝났다. 다음 회차에는 실제로 `supported_commands.h`에 `bitfield_ro`를 추가하고, 테스트를 통과시킨 뒤, week_5에서 만든 Minikube 환경에 재배포해서 `unknown command` 에러가 사라지는 것까지 확인해보자.

fin.
