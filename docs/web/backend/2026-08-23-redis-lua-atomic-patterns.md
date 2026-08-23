---
title: "Redis Lua 스크립트로 동시성 문제 풀기: 재고, 분산 락, Rate Limit"
date: 2026-08-23
tags: [redis, lua, concurrency, backend, distributed-lock]
description: "Lua 없이 짜면 깨지는 세 가지 실전 패턴을 통해 Redis 스크립트의 원자성이 무엇을 보장하고 무엇을 보장하지 않는지 정리한다."
---

## 학습 목적

Redis의 모든 명령은 원자적이다. 그런데도 재고가 음수로 떨어지고, 남의 락을 지우고, TTL 없는 키가 쌓인다.

명령 하나하나가 원자적이라는 사실이 **여러 명령을 묶은 로직**의 안전을 전혀 보장하지 않기 때문이다. 이 글은 그 틈이 실제로 어디서 벌어지는지를 세 가지 패턴으로 확인하고, Lua 스크립트가 그 틈을 어떻게 없애는지, 대신 무엇을 포기하는지를 정리한다.

Redis를 캐시로 쓰는 이야기는 [Redis 캐시 전략과 데이터 구조 활용](./2026-07-03-redis-cache-strategy.md)에서 다뤘다. 이 글은 Redis를 **상태 판단의 주체**로 쓸 때의 이야기다.

## 문제는 명령 사이의 빈틈이다

재고 차감을 애플리케이션 코드로 짜면 이렇게 된다.

```python
stock = redis.get("stock:1001")     # 1
if int(stock) >= 1:                  # 판단
    redis.decr("stock:1001")         # 차감
```

`GET`도 원자적이고 `DECR`도 원자적이다. 문제는 그 사이다.

```text
클라이언트 A              Redis(stock=1)           클라이언트 B
    │                         │                         │
    │──── GET stock ─────────▶│                         │
    │◀──────── 1 ─────────────│                         │
    │                         │◀──── GET stock ─────────│
    │                         │───────── 1 ────────────▶│
 "1개 남았네"                  │                    "1개 남았네"
    │──── DECR stock ────────▶│                         │
    │                         │  0                      │
    │                         │◀──── DECR stock ────────│
    │                         │  -1  ← 초과 판매        │
```

Redis는 잘못한 게 없다. 두 명령 모두 정확히 원자적으로 수행됐다. 잘못은 **판단을 클라이언트가 했다**는 데 있다. 판단하는 시점의 값과 쓰는 시점의 값이 다를 수 있다.

이건 RDB에서 갱신 유실(lost update)이라 부르는 것과 같은 구조의 문제다. 해결 방식도 대응된다 — [트랜잭션 격리 수준](../database/2026-08-14-transaction-isolation-levels.md)에서 정리한 비관적 락·낙관적 락·원자적 연산 중, Redis에서 주로 쓰는 것이 세 번째다.

## Redis가 스크립트에 주는 보장

Redis는 명령 실행이 단일 스레드다. 그리고 Lua 스크립트를 **명령 하나처럼** 취급한다. 공식 문서의 표현은 이렇다.

> Redis guarantees the script's atomic execution. While executing the script, all server activities are blocked during its entire runtime.

```text
클라이언트 A                        Redis
    │                          ┌─────────────────┐
    │── EVAL "GET·판단·DECR" ─▶│  스크립트 실행   │
    │                          │                 │  ← 이 구간에
클라이언트 B                     │   (분리 불가)    │     끼어들 수
    │── GET stock ────────────▶│                 │     없음
    │                          └─────────────────┘
    │◀───────────────────────────── (대기 후 응답)
```

즉 **락을 걸어서 경합을 관리하는 게 아니라, 경합할 구간 자체를 없앤다.** 대기는 Redis 서버가 알아서 시킨다.

## 패턴 1: 조건부 차감 — 재고, 포인트, 쿠폰

```lua
-- KEYS[1] = 재고 키, ARGV[1] = 차감 수량
local stock = redis.call('GET', KEYS[1])
if stock == false then
  return -1                                    -- 키 없음
end

local remain = tonumber(stock) - tonumber(ARGV[1])
if remain < 0 then
  return -2                                    -- 재고 부족
end

redis.call('DECRBY', KEYS[1], ARGV[1])
return remain                                  -- 차감 후 남은 수량
```

```bash
redis-cli SET stock:1001 1
redis-cli EVAL "$(cat decr_stock.lua)" 1 stock:1001 1
# (integer) 0     ← 성공, 0개 남음
redis-cli EVAL "$(cat decr_stock.lua)" 1 stock:1001 1
# (integer) -2    ← 재고 부족. DECRBY는 실행되지 않았다
```

`EVAL` 뒤의 `1`은 **키 개수**다. 이 숫자를 기준으로 앞쪽은 `KEYS`, 뒤쪽은 `ARGV`로 나뉜다. 이 구분은 단순한 관례가 아니다. Redis Cluster가 요청을 어느 노드로 보낼지 결정하는 근거이므로, **스크립트가 건드리는 키는 반드시 `KEYS`로 넘겨야 한다.**

### 반환값을 숫자로 설계하는 이유

Lua의 값은 Redis 응답으로 변환될 때 규칙이 있다. 특히 두 가지가 함정이다.

| Lua 값 | Redis 응답 |
| --- | --- |
| `true` | `1` |
| `false` | `nil` (null) |
| `3.7` | `3` (정수로 절삭) |
| 키가 없을 때 `redis.call('GET', ...)` | `false` |

`false`가 `nil`로 바뀌므로 **실패를 `false`로 반환하면 클라이언트에서 "값이 없음"과 구별되지 않는다.** 그래서 위 스크립트는 `-1`, `-2`처럼 실패 사유별로 다른 정수를 돌려준다. 어느 조건에서 걸렸는지 로그로 남길 수 있다는 점도 이득이다.

소수점이 잘린다는 점도 주의할 것. 금액을 다룬다면 원 단위 정수로 저장하거나 문자열로 반환해야 한다.

## 패턴 2: 분산 락 해제 — 남의 락을 지우는 사고

이 패턴이 Lua가 필요한 이유를 가장 선명하게 보여준다.

락을 잡는 것 자체는 명령 하나로 된다.

```bash
SET lock:order:1001 <내 고유 토큰> NX EX 30
```

`NX`(없을 때만)와 `EX`(만료)가 한 명령에 있으므로 안전하다. 문제는 **해제**다.

```python
# 위험한 코드
if redis.get("lock:order:1001") == my_token:
    redis.delete("lock:order:1001")
```

```text
시각  내 프로세스                    락 상태
 0s   SET ... NX EX 30 성공          내 토큰 (TTL 30s)
 …    작업이 30초를 넘김
30s                                  만료되어 사라짐
31s                                  다른 워커가 획득 → 남의 토큰
32s   GET → 어? 값이 있네            남의 토큰
      (내 토큰과 비교… 실패해야 정상)
```

여기까지는 비교에서 걸러진다. 진짜 사고는 비교와 삭제 **사이**에 만료가 일어날 때다.

```text
30.0s  GET → 내 토큰                 내 토큰 (TTL 0.01s 남음)
30.01s                               만료
30.02s                               다른 워커가 획득 → 남의 토큰
30.03s  DEL 실행                     ← 남의 락을 지웠다
```

내 락은 이미 사라졌는데, 나는 30.0초 시점의 판단으로 30.03초에 삭제를 실행했다. 결과적으로 **두 워커가 동시에 임계 구역에 들어간다.** 락을 쓰는 의미가 사라진다.

Lua로 묶으면 비교와 삭제 사이에 만료가 끼어들 수 없다.

```lua
-- KEYS[1] = 락 키, ARGV[1] = 내 고유 토큰
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

`ARGV`는 항상 문자열로 들어오므로 `GET` 결과와 그대로 비교된다. 토큰은 UUID처럼 워커마다 고유해야 한다. 프로세스 ID 같은 걸 쓰면 재시작 후 충돌한다.

### Redis 8.4부터는 명령 하나로 된다

공식 문서에 따르면 8.4에서 문자열 키에 대한 compare-and-set / compare-and-delete 명령이 추가됐다.

> Starting with version 8.4, Redis offers new atomic compare-and-set and compare-and-delete commands for string keys. … a client can use a single `DELEX` command for compare-and-delete: atomically deleting a string key if its value hasn't changed since it was fetched.

즉 이 패턴에 한해서는 `DELEX`로 대체할 수 있다. 다만 관리형 Redis는 버전이 뒤처지는 경우가 많으니 **쓰기 전에 실제 배포 버전을 확인해야 한다.** 조건 분기가 더 복잡해지면(예: 락 연장 + 카운터 갱신) 여전히 Lua가 필요하다.

## 패턴 3: Rate Limit — TTL 없는 좀비 키

고정 윈도우 방식의 요청 제한이다.

```python
# 위험한 코드
current = redis.incr(key)
if current == 1:
    redis.expire(key, 60)
```

`INCR`와 `EXPIRE` 사이에서 애플리케이션이 죽거나, 네트워크가 끊기거나, GC가 길게 돌면 **TTL이 없는 카운터가 남는다.** 이 키는 영원히 줄어들지 않으므로 해당 사용자는 영구 차단된다. 게다가 조용히 일어나서 알아채기 어렵다.

```lua
-- KEYS[1] = 카운터 키
-- ARGV[1] = 윈도우(초), ARGV[2] = 허용 횟수
local current = redis.call('INCR', KEYS[1])

if current == 1 then
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
end

if current > tonumber(ARGV[2]) then
  return 0                                  -- 차단
end
return ARGV[2] - current                    -- 남은 허용량
```

```bash
redis-cli EVAL "$(cat rate_limit.lua)" 1 rl:user:42 60 3
# (integer) 2   → 2   → 1   → 0(차단) → 0 …
```

남은 허용량을 돌려주면 `X-RateLimit-Remaining` 헤더로 바로 쓸 수 있다.

고정 윈도우는 **경계에서 최대 2배까지 통과한다.** 59초에 3개, 61초에 3개가 지나가면 2초 동안 6개다. 이걸 막으려면 Sorted Set 기반 슬라이딩 윈도우나 토큰 버킷으로 가야 하는데, 둘 다 "현재 시각 조회 → 오래된 항목 제거 → 개수 확인 → 추가"라는 다단계 로직이라 **Lua가 더더욱 필요해진다.**

## 세 패턴을 관통하는 구조

```text
1. 읽는다      redis.call('GET' | 'INCR' | ...)
2. 판단한다    if ... then          ← 클라이언트가 하면 깨지는 지점
3. 쓴다        redis.call('DECRBY' | 'DEL' | 'EXPIRE' | ...)
4. 알린다      return <정수 코드>
```

**2번을 서버로 옮기는 것**이 전부다. 반대로 말하면, 판단 없이 쓰기만 한다면 Lua는 필요 없다. `MULTI`/`EXEC`나 명령 하나로 충분하다.

## 애플리케이션에서 호출하기

스크립트를 코드 안에 문자열로 박아 두면 문법 강조도 안 되고 diff도 읽기 어렵다. **별도 `.lua` 파일로 분리해 버전 관리**하는 편이 낫다.

```text
app/
├── redis_scripts/
│   ├── decr_stock.lua
│   ├── release_lock.lua
│   └── rate_limit.lua
└── inventory.py
```

### Python (redis-py)

`register_script()`가 돌려주는 객체는 호출 가능하고, **`EVALSHA` 캐싱과 `NOSCRIPT` 폴백을 알아서 처리한다.**

```python
from pathlib import Path
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

SCRIPTS = Path(__file__).parent / "redis_scripts"
decr_stock = r.register_script((SCRIPTS / "decr_stock.lua").read_text())


class OutOfStock(Exception):
    pass


class ProductNotFound(Exception):
    pass


def purchase(product_id: int, quantity: int) -> int:
    """차감에 성공하면 남은 재고를 반환한다."""
    result = decr_stock(keys=[f"stock:{product_id}"], args=[quantity])

    if result == -1:
        raise ProductNotFound(product_id)
    if result == -2:
        raise OutOfStock(product_id)
    return result
```

`keys`와 `args`를 **별도 인자로 나눠 넘기는 것**에 주목할 것. redis-py가 `len(keys)`를 계산해 `EVALSHA`의 키 개수 자리에 넣어 준다. 직접 세어 넘길 일이 없다.

정수 코드를 그대로 흘려보내지 않고 예외로 바꾸는 것도 중요하다. `-2`가 호출부까지 올라가면 결국 어딘가에서 매직 넘버를 비교하게 된다.

### 폴백은 실제로 이렇게 동작한다

redis-py의 `Script.__call__` 구현이다.

```python
try:
    return client.evalsha(self.sha, len(keys), *args)
except NoScriptError:
    # 서버가 재시작해 캐시가 비었을 수 있다
    self.sha = client.script_load(self.script)
    return client.evalsha(self.sha, len(keys), *args)
```

`NOSCRIPT`를 잡아 다시 올리고 재시도한다. 앞서 말한 폴백이 그대로 들어 있다.

다만 **클러스터 파이프라인에서는 예외**다. 같은 코드의 위쪽에 이런 분기가 있다.

```python
if isinstance(client, ClusterPipeline):
    # ClusterPipeline does not support script_load. Queue EVALSHA and
    # leave NOSCRIPT recovery to the caller
    return client.evalsha(self.sha, len(keys), *args)
```

파이프라인 안에서는 `NOSCRIPT` 복구를 **호출자가 직접 해야 한다.** Redis 공식 문서가 파이프라인 컨텍스트에서는 `EVAL`을 쓰라고 하는 것과 같은 이야기다.

### Node.js (ioredis)

`defineCommand`로 등록하면 **커스텀 명령처럼** 쓸 수 있다.

```javascript
import Redis from "ioredis";
import { readFileSync } from "fs";
import { randomUUID } from "crypto";

const redis = new Redis();

redis.defineCommand("releaseLock", {
  numberOfKeys: 1,
  lua: readFileSync("./redis_scripts/release_lock.lua", "utf8"),
});

async function withLock(key, ttlSec, fn) {
  const token = randomUUID();

  const acquired = await redis.set(key, token, "NX", "EX", ttlSec);
  if (acquired !== "OK") return false;

  try {
    return await fn();
  } finally {
    // 내 토큰일 때만 지운다. 만료 후 남이 잡은 락은 건드리지 않는다
    await redis.releaseLock(key, token);
  }
}
```

`numberOfKeys: 1`을 선언해 뒀으므로 호출할 때 키 개수를 넘기지 않는다. ioredis도 내부적으로 `EVALSHA`를 우선 쓴다.

`finally`에서 해제하는 것이 핵심이다. 예외가 나도 락은 풀려야 한다. 다만 **작업이 TTL보다 오래 걸리면 `finally`가 돌기 전에 이미 만료**되므로, 그때 남의 락을 지우지 않게 막아 주는 것이 바로 이 스크립트다.

### 스크립트를 배포할 때

| 방식 | 동작 | 트레이드오프 |
| --- | --- | --- |
| 앱 시작 시 `SCRIPT LOAD` | 배포 시점에 한 번 올림 | 서버 재시작·복제본 승격 시 캐시가 비어 폴백 필요 |
| 클라이언트 라이브러리에 위임 | 첫 호출 때 자동 등록 | 가장 단순. 대부분 이걸로 충분하다 |
| Redis Functions (7.0+) | 서버에 영속되는 라이브러리로 등록 | 앱과 분리돼 관리·버전 관리가 쉽지만 배포 절차가 하나 늘어난다 |

스크립트가 몇 개뿐이면 두 번째로 충분하다. 스크립트가 수십 개가 되고 여러 서비스가 공유하기 시작하면 Functions를 검토할 시점이다.

## 운영에서 밟는 지뢰

### EVALSHA와 NOSCRIPT

매번 스크립트 본문을 보내면 대역폭이 낭비된다. `SCRIPT LOAD`로 올리고 SHA1 해시로 호출한다.

```bash
redis-cli SCRIPT LOAD "$(cat decr_stock.lua)"
# "c664a3bf70bd1d45c4284ffebb65a6f2299bfc9f"
redis-cli EVALSHA c664a3bf... 1 stock:1001 1
```

문제는 **스크립트 캐시가 영속되지 않는다**는 점이다. 문서의 표현은 "always volatile"이고, 서버 재시작·복제본 승격·`SCRIPT FLUSH`로 비워진다. 즉 `EVALSHA`는 언제든 실패할 수 있다.

```text
(error) NOSCRIPT No matching script
```

폴백 처리는 앞서 본 대로 주요 클라이언트에 내장돼 있다. 여기서 기억할 것은 **왜 비워지는가**다. 배포 직후가 아니라 **복제본 승격이 일어난 새벽에** `NOSCRIPT`가 터진다. 장애 대응 중에 처음 보면 원인을 찾기 어렵다.

### 원자성은 롤백이 아니다

가장 오해하기 쉬운 지점이다. Redis는 롤백을 지원하지 않는다.

> Redis does not support rollbacks of transactions since supporting rollbacks would have a significant impact on the simplicity and performance of Redis.

스크립트 중간에 에러가 나면 **이미 실행된 쓰기는 그대로 남는다.** 원자성이 보장하는 것은 "다른 클라이언트가 중간 상태를 보지 못한다"는 것이지, "실패하면 없던 일이 된다"가 아니다.

```lua
redis.call('DECRBY', KEYS[1], 1)      -- 실행됨
redis.call('LPUSH', KEYS[2], 'x')     -- 여기서 WRONGTYPE 에러
-- 위의 DECRBY는 되돌아가지 않는다
```

그래서 스크립트는 **검증을 모두 끝낸 뒤에 쓰기를 시작하는 순서**로 짜야 한다. 패턴 1에서 `-1`, `-2` 반환이 모두 `DECRBY` 앞에 있는 이유가 이것이다.

### 긴 스크립트는 서버를 세운다

스크립트에는 최대 실행 시간이 있다. 설정 이름은 `busy-reply-threshold`이고 기본값은 5초다. 그런데 이 값을 넘겨도 **Redis가 스크립트를 죽이지 않는다.** 죽이면 원자성 계약이 깨지기 때문이다.

대신 이렇게 된다.

| 상황 | Redis의 반응 |
| --- | --- |
| 임계값 초과 | 로그를 남기고, 다른 클라이언트에 `BUSY` 에러 응답 시작 |
| 이 상태에서 허용되는 명령 | `SCRIPT KILL`, `FUNCTION KILL`, `SHUTDOWN NOSAVE` 뿐 |
| 스크립트가 **읽기만** 했다면 | `SCRIPT KILL`로 중단 가능 |
| 스크립트가 **한 번이라도 썼다면** | `SHUTDOWN NOSAVE` 외에 방법 없음 |

마지막 줄이 핵심이다. **쓰기를 한 스크립트가 무한 루프에 빠지면, 유일한 탈출구는 저장 없이 서버를 내리는 것이다.** 그 순간 마지막 스냅샷 이후의 데이터가 날아간다.

그래서 스크립트에는 **루프를 넣지 않는 것이 원칙**이다. `KEYS *`나 큰 컬렉션 전체 순회도 같은 이유로 금지다. 세 패턴이 모두 분기만 있고 반복이 없는 것은 우연이 아니다.

### 클러스터에서의 제약

문서에 따르면 `#!` 셔뱅 없는 스크립트는 서로 다른 해시 슬롯의 키에 접근할 수 있지만, 셔뱅이 있는 스크립트는 기본 플래그를 상속받아 그럴 수 없다. 어느 쪽이든 **같은 슬롯에 모으는 것이 안전하다.** 해시 태그를 쓴다.

```text
{order:1001}:stock     ┐
{order:1001}:lock      ├─ 중괄호 안이 같으므로 같은 슬롯
{order:1001}:coupon    ┘
```

## 자주 하는 오해

**"Lua를 쓰면 성능이 좋아진다"** — 네트워크 왕복이 줄어드는 만큼은 맞다. 하지만 스크립트가 도는 동안 서버 전체가 멈추므로, 무거운 스크립트는 전체 처리량을 오히려 떨어뜨린다. Lua는 성능 최적화 도구가 아니라 **정확성 도구**다.

**"Lua 스크립트는 트랜잭션이다"** — 원자성과 격리는 있지만 롤백이 없다. ACID의 A를 "all or nothing"으로 이해하고 있다면 그 의미가 아니다.

**"`MULTI`/`EXEC`로도 되지 않나"** — `MULTI`는 명령을 큐에 모아 한 번에 실행할 뿐이고, **중간 결과로 분기할 수 없다.** "재고가 있으면 차감"처럼 값을 보고 판단해야 하면 쓸 수 없다. 공식 문서도 "Everything you can do with a Redis Transaction, you can also do with a script, and usually the script will be both simpler and faster"라고 적고 있다.

**"단일 스레드니까 느리다"** — 락 경합과 컨텍스트 스위칭이 없어 인메모리 연산에는 오히려 유리하다. 파이썬 GIL이 CPU 바운드에서 불리한 것과는 상황이 다르다. 관련 논의는 [Python GIL과 동시성](../../language/2026-07-17-python-gil-concurrency.md)에 정리해 뒀다.

## 정리

- 명령 하나하나의 원자성은 **여러 명령을 묶은 로직**의 안전을 보장하지 않는다. 문제는 항상 판단과 쓰기 사이의 틈이다.
- Lua 스크립트는 그 구간을 서버로 옮겨 쪼개지지 않게 만든다. 락이 아니라 **경합 구간 제거**다.
- 세 패턴 모두 구조가 같다 — 읽고, **판단하고**, 쓰고, 정수 코드로 알린다.
- 실패 사유는 `false`가 아니라 서로 다른 음수로 반환한다. Lua의 `false`는 Redis에서 `nil`이 되어 "값 없음"과 구별되지 않는다.
- **원자성은 롤백이 아니다.** 에러가 나도 이미 실행된 쓰기는 남는다. 검증을 모두 끝낸 뒤 쓰기를 시작해야 한다.
- 스크립트는 짧고 반복이 없어야 한다. 쓰기를 한 스크립트가 멈추면 `SHUTDOWN NOSAVE` 외에 방법이 없다.
- `EVALSHA`는 `NOSCRIPT` 폴백이 필수다. 스크립트 캐시는 영속되지 않는다.

## 학습 체크리스트

- [ ] `GET` 후 `DECR`이 왜 안전하지 않은지 타임라인으로 설명할 수 있는가
- [ ] 분산 락 해제에서 `GET` 후 `DEL`이 남의 락을 지우는 시나리오를 재현해 설명할 수 있는가
- [ ] `KEYS`와 `ARGV`를 나누는 이유가 클러스터와 어떤 관련이 있는지 설명할 수 있는가
- [ ] Lua에서 `false`를 반환하면 안 되는 이유를 설명할 수 있는가
- [ ] Redis의 원자성과 RDB 트랜잭션의 원자성이 어떻게 다른지 설명할 수 있는가
- [ ] 쓰기를 수행한 스크립트가 무한 루프에 빠졌을 때 무엇을 할 수 있는지 아는가
- [ ] `MULTI`/`EXEC`로 풀 수 있는 문제와 없는 문제를 구분할 수 있는가
- [ ] 클라이언트 라이브러리가 `NOSCRIPT`를 어떻게 처리하는지, 그 처리가 안 되는 경우는 언제인지 아는가
- [ ] 스크립트의 정수 반환값을 애플리케이션 예외로 바꿔야 하는 이유를 설명할 수 있는가

## 참고

- [Scripting with Lua — Redis Docs](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis programmability — Maximum execution time](https://redis.io/docs/latest/develop/programmability/)
- [Transactions — Redis Docs](https://redis.io/docs/latest/develop/using-commands/transactions/)
- [Redis Lua API Reference](https://redis.io/docs/latest/develop/programmability/lua-api)
- [Distributed Locks with Redis](https://redis.io/docs/latest/develop/use-cases/patterns/distributed-locks/)
