---
title: "Redis 캐시 전략과 데이터 구조 활용"
date: 2026-07-03
tags: [redis, cache, backend, performance, database]
description: "Redis의 Look-aside, Write-through 캐시 전략과 데이터 구조 활용, Cache Stampede 해결책을 정리한다."
---

## 학습 목적

Redis는 단순히 빠른 key-value 저장소가 아니라, 서비스 성능을 높이기 위한 캐시 계층으로 자주 사용된다. 특히 읽기 부하가 큰 API, 세션 관리, 랭킹, 실시간 카운터, 작업 큐 같은 기능에서 효과적이다.

이번 글에서는 Redis를 사용할 때 자주 나오는 캐시 전략과 데이터 구조 활용 방식, 그리고 캐시가 동시에 만료될 때 발생하는 Cache Stampede 해결책을 정리한다.

## Redis를 캐시로 사용하는 이유

Redis는 메모리 기반 저장소라서 디스크 기반 DB보다 빠르게 데이터를 읽고 쓸 수 있다. 그래서 자주 조회되는 데이터를 Redis에 저장해두면 DB 조회 횟수를 줄이고 응답 속도를 개선할 수 있다.

다만 Redis를 캐시로 사용할 때는 아래 내용을 함께 고려해야 한다.

- 원본 데이터는 어디에 있는가
- 캐시 데이터는 언제 갱신되는가
- 캐시가 비었을 때 누가 다시 채우는가
- 캐시 데이터가 오래되어도 허용 가능한가
- Redis 장애 시 서비스가 어떻게 동작해야 하는가

## Look-aside 전략

Look-aside는 Cache-aside라고도 부른다. 애플리케이션이 먼저 Redis를 조회하고, 캐시에 없으면 DB를 조회한 뒤 Redis에 다시 저장하는 방식이다.

흐름은 아래와 같다.

1. 클라이언트가 API를 호출한다.
2. 서버가 Redis에서 캐시 데이터를 조회한다.
3. 캐시에 있으면 바로 응답한다.
4. 캐시에 없으면 DB를 조회한다.
5. DB 조회 결과를 Redis에 저장한다.
6. 클라이언트에 응답한다.

```text
Request
  -> Redis GET
  -> cache hit: return cached data
  -> cache miss: DB SELECT
  -> Redis SET with TTL
  -> return DB data
```

예시는 아래와 같다.

```js
async function getUser(userId) {
  const cacheKey = `user:${userId}`
  const cachedUser = await redis.get(cacheKey)

  if (cachedUser) {
    return JSON.parse(cachedUser)
  }

  const user = await userRepository.findById(userId)
  await redis.set(cacheKey, JSON.stringify(user), { EX: 60 })

  return user
}
```

### 장점

- 실제로 조회된 데이터만 캐시에 저장하므로 메모리를 효율적으로 쓴다.
- 애플리케이션에서 캐시 정책을 직접 제어하기 쉽다.
- 읽기 요청이 많은 서비스에 적용하기 쉽다.

### 단점

- 최초 요청은 항상 캐시 miss가 발생한다.
- 데이터 변경 시 캐시 삭제 또는 갱신 처리를 직접 해야 한다.
- 인기 key가 동시에 만료되면 DB에 요청이 몰릴 수 있다.

## Write-through 전략

Write-through는 데이터를 쓸 때 DB와 Redis를 함께 갱신하는 방식이다. 읽기 시점에 캐시를 채우는 Look-aside와 달리, 쓰기 시점에 캐시를 최신 상태로 맞춘다.

흐름은 아래와 같다.

1. 클라이언트가 데이터 변경 요청을 보낸다.
2. 서버가 DB를 갱신한다.
3. 서버가 Redis 캐시도 함께 갱신한다.
4. 이후 읽기 요청은 Redis에서 최신 데이터를 읽는다.

```text
Write Request
  -> DB UPDATE
  -> Redis SET
  -> return success
```

예시는 아래와 같다.

```js
async function updateUserName(userId, name) {
  const user = await userRepository.updateName(userId, name)

  await redis.set(`user:${userId}`, JSON.stringify(user), { EX: 60 })

  return user
}
```

### 장점

- 캐시가 비교적 최신 상태를 유지한다.
- 읽기 요청에서 cache miss가 줄어든다.
- 자주 읽히는 데이터의 응답 속도를 안정적으로 유지할 수 있다.

### 단점

- 쓰기 작업마다 Redis 갱신이 추가되어 write latency가 늘어난다.
- DB 갱신은 성공했지만 Redis 갱신이 실패하는 상황을 처리해야 한다.
- 실제로 읽히지 않는 데이터까지 캐시에 저장될 수 있다.

## Look-aside와 Write-through 비교

| 구분 | Look-aside | Write-through |
| --- | --- | --- |
| 캐시 저장 시점 | 읽기 miss 발생 시 | 쓰기 발생 시 |
| 메모리 효율 | 높음 | 상대적으로 낮음 |
| 읽기 성능 | warm-up 이후 좋음 | 처음부터 좋음 |
| 쓰기 성능 | 캐시 삭제 정도로 가볍게 처리 가능 | DB와 Redis를 함께 갱신해야 함 |
| 데이터 최신성 | TTL과 invalidation에 의존 | 상대적으로 최신 상태 유지 |
| 적합한 경우 | 읽기 중심, 일부 데이터만 자주 조회 | 읽기 빈도가 높고 최신성이 중요한 데이터 |

실무에서는 두 전략을 섞어 쓰는 경우가 많다. 예를 들어 조회 API는 Look-aside로 구성하고, 데이터 수정 API에서는 관련 cache key를 삭제하거나 최신 값으로 갱신한다.

## Redis 데이터 구조 활용

Redis는 문자열만 저장하는 도구가 아니다. 데이터 구조를 잘 고르면 애플리케이션 코드와 DB 부하를 줄일 수 있다.

### String

가장 기본적인 구조다. JSON 직렬화 데이터, 토큰, 단순 카운터에 사용한다.

```text
SET user:1 '{"id":1,"name":"Astar"}' EX 60
INCR article:1:view_count
```

적합한 경우:

- API 응답 캐시
- 인증 토큰
- 단순 카운터
- feature flag

### Hash

하나의 key 안에 field-value를 저장한다. 객체의 일부 필드만 읽거나 수정할 때 유용하다.

```text
HSET user:1 name "Astar" level "junior"
HGET user:1 name
HINCRBY user:1 login_count 1
```

적합한 경우:

- 사용자 프로필
- 설정 값 묶음
- 장바구니 요약 정보

### List

순서가 있는 목록이다. 앞뒤로 데이터를 넣고 빼는 작업에 적합하다.

```text
LPUSH recent:articles 100
LRANGE recent:articles 0 9
```

적합한 경우:

- 최근 본 상품
- 최근 작성 글
- 간단한 queue

### Set

중복 없는 집합이다. membership check와 교집합, 합집합 같은 연산에 좋다.

```text
SADD article:1:likes user:1
SISMEMBER article:1:likes user:1
SCARD article:1:likes
```

적합한 경우:

- 좋아요 사용자 목록
- 태그별 게시글 ID
- 중복 제거
- 권한 또는 그룹 membership 확인

### Sorted Set

각 값에 score를 부여하고 score 기준으로 정렬한다.

```text
ZADD ranking 1500 user:1
ZREVRANGE ranking 0 9 WITHSCORES
```

적합한 경우:

- 랭킹
- 인기 게시글
- 우선순위 큐
- 시간순 피드

### Stream

append-only log 형태의 데이터 구조다. 이벤트를 순서대로 기록하고 consumer group으로 처리할 수 있다.

```text
XADD order:events * orderId 1 status paid
XREAD COUNT 10 STREAMS order:events 0
```

적합한 경우:

- 이벤트 로그
- 비동기 작업 처리
- 간단한 메시지 스트림
- audit trail

## Cache Stampede

Cache Stampede는 인기 있는 캐시 key가 만료되는 순간 여러 요청이 동시에 DB로 몰리는 현상이다.

예를 들어 `article:popular:today` key가 만료되었을 때 동시에 1,000개의 요청이 들어오면, 모든 요청이 Redis miss를 보고 DB를 조회할 수 있다. 이 경우 캐시를 쓰고 있는데도 DB 부하가 순간적으로 폭증한다.

## Cache Stampede 해결책

### 1. Mutex Lock

캐시 miss가 발생했을 때 하나의 요청만 DB를 조회하고 캐시를 채우도록 lock을 건다.

```text
SET lock:article:popular:today random-value NX PX 3000
```

lock을 얻은 요청만 DB를 조회하고 Redis를 갱신한다. 다른 요청은 잠깐 대기하거나 기존 값을 반환한다.

주의할 점:

- lock에는 반드시 TTL을 둔다.
- lock 해제 시 본인이 만든 lock인지 확인해야 한다.
- lock 시간이 너무 길면 응답 지연이 커지고, 너무 짧으면 중복 조회가 발생할 수 있다.

### 2. TTL Jitter

캐시 key들의 만료 시간이 동시에 몰리지 않도록 TTL에 랜덤 값을 섞는다.

```js
const baseTtl = 300
const jitter = Math.floor(Math.random() * 60)

await redis.set(cacheKey, value, { EX: baseTtl + jitter })
```

예를 들어 모든 key를 정확히 5분 뒤 만료시키는 대신 5분에서 6분 사이에 분산시키면 동시에 많은 key가 만료되는 문제를 줄일 수 있다.

### 3. Stale-while-revalidate

캐시가 조금 오래되었더라도 일단 기존 값을 반환하고, 백그라운드에서 새 값을 갱신하는 방식이다.

적합한 경우:

- 약간 오래된 데이터가 허용되는 화면
- 인기 게시글 목록
- 통계성 데이터
- 추천 목록

부적합한 경우:

- 결제 상태
- 재고 차감
- 권한 확인
- 잔액 조회

### 4. Cache Pre-warming

트래픽이 몰리기 전에 자주 조회될 데이터를 미리 캐시에 넣는 방식이다.

적합한 경우:

- 메인 화면 데이터
- 이벤트 페이지
- 랭킹
- 카테고리 목록
- 배포 직후 반드시 필요한 설정 데이터

### 5. Hot Key 분리

특정 key 하나에 트래픽이 몰리면 Redis 자체에도 부담이 된다. 이런 경우 key를 나누거나, 로컬 메모리 캐시를 함께 사용할 수 있다.

예시는 아래와 같다.

```text
article:popular:today:0
article:popular:today:1
article:popular:today:2
```

단, key를 나누면 정합성 관리가 어려워질 수 있으므로 정말 hot key인지 지표로 확인한 뒤 적용해야 한다.

## 운영 시 확인할 지표

Redis 캐시를 운영할 때는 아래 지표를 확인해야 한다.

- cache hit ratio
- Redis memory usage
- evicted keys
- expired keys
- hot key 여부
- DB query count 감소 여부
- P95/P99 latency
- Redis command latency
- connection count

캐시 hit ratio만 높다고 좋은 것은 아니다. 오래된 데이터를 반환하거나, 캐시 갱신 비용이 너무 크면 전체 서비스 품질은 나빠질 수 있다.

## 정리

- Look-aside는 읽기 시점에 캐시를 채우는 전략이다.
- Write-through는 쓰기 시점에 DB와 캐시를 함께 갱신하는 전략이다.
- Redis 데이터 구조는 사용 목적에 맞게 골라야 한다.
- Cache Stampede는 인기 key 만료 시 DB 요청이 폭증하는 문제다.
- Stampede는 mutex lock, TTL jitter, stale-while-revalidate, pre-warming으로 완화할 수 있다.

Redis는 빠른 저장소이지만, 캐시 정책 없이 붙이면 오히려 장애 지점을 하나 더 만드는 결과가 될 수 있다. 어떤 데이터가 얼마나 오래 stale해도 되는지, 캐시가 실패했을 때 DB로 fallback할 수 있는지까지 함께 설계해야 한다.

## 참고

- [Redis cache-aside](https://redis.io/docs/latest/develop/use-cases/cache-aside/)
- [Redis data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Distributed locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
