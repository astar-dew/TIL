---
title: "서버리스 DB의 트랜잭션 처리: Cloudflare D1과 카카오페이 결제"
date: 2026-07-30
tags: [cloudflare, d1, serverless, transaction, payment, kakaopay, aws, sqlite]
description: "Cloudflare D1이 제공하는 트랜잭션 수단과 한계를 정리하고, 카카오페이 결제 흐름을 D1 위에서 안전하게 구현하는 방법 및 AWS 대비 성능·비용 차이를 비교한다."
---

## 학습 목적

Cloudflare Workers 위에서 D1을 사용해 카카오페이 결제를 붙이려고 하면 가장 먼저 막히는 지점이 트랜잭션이다. 기존 RDB 감각으로는 `BEGIN` ... `COMMIT` 사이에 결제 로직을 넣고 싶은데, D1에는 그 API 자체가 없다.

이 글에서는 세 가지를 정리한다.

1. D1이 트랜잭션을 어떤 방식으로 제공하고, 무엇을 제공하지 않는지
2. 그 제약 위에서 카카오페이 결제를 어떻게 구현해야 하는지
3. AWS(RDS/Aurora/DynamoDB) 대비 성능·비용 관점에서 어떤 차이가 있는지

---

## 먼저 짚을 것: 결제는 DB 트랜잭션 문제가 아니다

D1의 트랜잭션 제약을 논하기 전에 전제를 하나 정리해야 한다.

카카오페이 결제 흐름은 다음과 같다.

```text
1. 결제 준비   POST /online/v1/payment/ready    → tid, next_redirect_url 수신
2. 사용자 인증  사용자가 카카오페이 앱/웹에서 승인 → approval_url로 리다이렉트(pg_token 전달)
3. 결제 승인   POST /online/v1/payment/approve  → 최종 승인, aid 발급
4. 취소/조회   POST /online/v1/payment/cancel, /online/v1/payment/order
```

2번은 사람이 개입하는 단계다. 수 초에서 수 분이 걸리고, 사용자가 창을 닫아버릴 수도 있다. 즉 1번과 3번 사이에는 **어떤 DB 트랜잭션도 열어둘 수 없다.** 이건 D1이 아니라 PostgreSQL을 써도 마찬가지다.

그리고 3번의 결제 승인은 우리 DB가 아니라 카카오페이 서버에서 확정된다. 외부 HTTP 호출은 트랜잭션 롤백으로 되돌릴 수 없다. `approve` 요청이 성공했는데 우리 DB 커밋이 실패하면, DB를 롤백해도 사용자 돈은 이미 빠져나간 상태다.

```text
[잘못된 설계]
BEGIN
  UPDATE orders SET status = 'PAID'
  await kakaoPayApprove()   ← 여기서 네트워크 타임아웃
COMMIT                       ← 롤백해도 결제는 이미 승인됨
```

정리하면 결제는 **DB 로컬 트랜잭션 문제가 아니라 분산 트랜잭션(saga) 문제**다. 따라서 D1이 대화형 트랜잭션을 지원하지 않는다는 사실은, 결제 도메인에서는 생각보다 큰 손해가 아니다. 진짜로 필요한 원자성의 범위는 "우리 DB 안에서 여러 행을 한 번에 바꾸는 것"으로 좁혀지고, 그건 D1도 제공한다.

---

## D1이 트랜잭션을 다루는 방식

### 아키텍처 전제

D1은 SQLite를 Durable Objects 스토리지 위에 올린 구조다. 여기서 나오는 두 가지 특성이 트랜잭션 모델을 거의 결정한다.

- **데이터베이스 하나당 쓰기 primary는 하나**다. 읽기는 복제본으로 분산할 수 있지만 쓰기는 항상 primary로 간다.
- **각 데이터베이스는 단일 스레드로 동작한다.** 공식 문서 표현으로 "inherently single-threaded, and processes queries one at a time"이다.

즉 D1에는 동시 쓰기 경합이 존재하지 않는다. 쓰기는 primary에서 순차 처리되므로, MVCC나 격리 수준 같은 개념이 애초에 노출될 필요가 없다.

### 기본 동작은 auto-commit

D1은 auto-commit 모드로 동작한다. `prepare().run()` 하나가 곧 하나의 트랜잭션이고, 실행 즉시 커밋된다.

```js
const result = await env.DB
  .prepare("UPDATE orders SET status = ?1 WHERE id = ?2")
  .bind("PAID", orderId)
  .run();

// result.meta 에서 실행 정보를 확인할 수 있다
// changes, last_row_id, duration, rows_read, rows_written
```

### 제공하는 것: `batch()` — 암묵적 트랜잭션

D1이 제공하는 유일한 다중 문장 원자성 수단은 `batch()`다.

```js
const results = await env.DB.batch([
  env.DB.prepare("UPDATE orders SET status = ?1 WHERE id = ?2")
        .bind("PAID", orderId),
  env.DB.prepare(
    "INSERT INTO payments (order_id, tid, aid, amount, approved_at) VALUES (?1, ?2, ?3, ?4, ?5)"
  ).bind(orderId, tid, aid, amount, approvedAt),
  env.DB.prepare("INSERT INTO payment_events (order_id, type, payload) VALUES (?1, ?2, ?3)")
        .bind(orderId, "APPROVED", JSON.stringify(kakaoResponse)),
]);
```

공식 문서는 이렇게 명시한다.

> "Batched statements are SQL transactions. If a statement in the sequence fails, then an error is returned for that specific statement, and it aborts or rolls back the entire sequence."

동작 특성은 다음과 같다.

- 배열 안의 문장들은 **순차적으로, 동시성 없이** 실행된다.
- 하나라도 실패하면 **전체가 롤백**된다.
- 네트워크 왕복은 **1회**다. 문장 5개를 개별 `run()`으로 보내면 왕복 5회지만 `batch()`는 1회다. 이건 성능상 꽤 큰 차이다.

### 제공하지 않는 것

| 기능 | D1 | 비고 |
|---|---|---|
| `BEGIN` / `COMMIT` / `ROLLBACK` 직접 실행 | ✗ | Worker API로 실행 불가 |
| 대화형 트랜잭션(읽고 → 판단하고 → 쓰기를 한 트랜잭션 안에서) | ✗ | `batch()`는 문장이 미리 확정되어야 함 |
| `SAVEPOINT` / 중첩 트랜잭션 | ✗ | |
| 격리 수준 선택 | ✗ | 단일 스레드라 선택지가 없음 |
| `SELECT ... FOR UPDATE` 등 명시적 락 | ✗ | 동시 쓰기가 없으므로 개념 자체가 없음 |
| 배치 중간 결과를 다음 문장에 전달 | ✗ | SQL 안에서 서브쿼리로 해결해야 함 |

**왜 지원하지 않는가.** Worker와 D1은 네트워크로 분리되어 있다. 대화형 트랜잭션을 허용하면 애플리케이션이 왕복하는 동안 단일 스레드 DB가 락을 잡고 대기한다. 클라이언트가 죽으면 트랜잭션이 무기한 열린 채로 남아 DB 전체가 멈춘다. 서버리스 환경에서는 이 위험을 감당할 수 없기 때문에 구조적으로 배제한 선택에 가깝다.

### 읽기 복제와 Sessions API

D1은 읽기 복제본(read replication)을 제공하고, `withSession()`으로 순차 일관성(sequential consistency)을 보장한다.

```js
// 쓰기 직후의 읽기는 반드시 primary 또는 bookmark 기반으로
const session = env.DB.withSession("first-primary");

const order = await session
  .prepare("SELECT status FROM orders WHERE id = ?1")
  .bind(orderId)
  .first();

// 이후 요청에 이어붙일 수 있도록 bookmark를 저장
const bookmark = session.getBookmark();
```

결제에서 이게 중요한 이유가 있다. `approve` 성공 후 상태를 `PAID`로 쓰고, 곧바로 주문 조회 API가 호출되면 복제 지연 때문에 아직 `READY`로 보일 수 있다. 결제 완료 화면에서 "결제 대기 중"이 뜨는 버그가 여기서 나온다.

- 결제 관련 읽기는 `first-primary` 또는 이전 쓰기의 bookmark를 사용한다.
- bookmark는 쿠키나 응답 헤더에 실어 클라이언트 후속 요청으로 전달하는 패턴이 일반적이다.

### Time Travel

D1은 30일(유료 플랜 기준) 시점 복구를 제공한다. 특정 타임스탬프나 bookmark로 되감을 수 있다.

```bash
wrangler d1 time-travel restore <DB_NAME> --timestamp=2026-07-30T09:00:00Z
```

다만 이건 재해 복구 수단이지 트랜잭션 롤백 대체재가 아니다. 결제 DB를 통째로 과거로 되돌리면 그 사이의 정상 결제까지 사라진다.

---

## D1 위에서 카카오페이 결제 구현하기

### 원칙

1. 외부 PG 호출은 절대 원자적 단위 안에 넣지 않는다.
2. 로컬 상태 전이는 조건부 `UPDATE`(compare-and-swap)로 처리한다.
3. 모든 외부 호출은 멱등키를 갖는다.
4. 진실의 원천(source of truth)은 카카오페이다. 우리 DB는 미러이고, 대사(reconciliation)로 맞춘다.

### 스키마

```sql
CREATE TABLE orders (
  id             TEXT PRIMARY KEY,
  user_id        TEXT NOT NULL,
  amount         INTEGER NOT NULL,
  status         TEXT NOT NULL,  -- CREATED, READY, APPROVING, PAID, FAILED, CANCELED
  created_at     INTEGER NOT NULL,
  updated_at     INTEGER NOT NULL
);

CREATE TABLE payments (
  order_id       TEXT PRIMARY KEY REFERENCES orders(id),
  tid            TEXT NOT NULL UNIQUE,   -- 카카오페이 거래 ID
  aid            TEXT UNIQUE,            -- 승인 번호 (승인 후 채워짐)
  amount         INTEGER NOT NULL,
  approved_at    INTEGER
);

-- 승인 요청 멱등성 보장용
CREATE TABLE payment_attempts (
  idempotency_key TEXT PRIMARY KEY,      -- tid + pg_token 해시
  order_id        TEXT NOT NULL,
  result          TEXT,                  -- 성공 시 응답 JSON 캐시
  created_at      INTEGER NOT NULL
);

CREATE TABLE payment_events (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  order_id   TEXT NOT NULL,
  type       TEXT NOT NULL,
  payload    TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE INDEX idx_orders_status_updated ON orders(status, updated_at);
CREATE INDEX idx_payment_events_order   ON payment_events(order_id, created_at);
```

`payments.tid`와 `payment_attempts.idempotency_key`의 `UNIQUE` 제약이 중복 승인을 DB 레벨에서 막는 마지막 방어선이다.

### 핵심 기법: 조건부 UPDATE로 상태 전이 선점

대화형 트랜잭션이 없으면 "읽어서 확인하고 → 쓰기"를 원자적으로 못 한다. 대신 **판단 조건을 SQL의 `WHERE` 절 안에 넣는다.**

```js
// ✗ 나쁜 예: 읽기와 쓰기 사이에 다른 요청이 끼어들 수 있다
const order = await env.DB.prepare("SELECT status FROM orders WHERE id = ?1")
  .bind(orderId).first();
if (order.status !== "READY") throw new Error("invalid state");
await env.DB.prepare("UPDATE orders SET status = 'APPROVING' WHERE id = ?1")
  .bind(orderId).run();

// ✓ 좋은 예: 조건과 갱신을 한 문장으로 (compare-and-swap)
const claim = await env.DB
  .prepare(`UPDATE orders
               SET status = 'APPROVING', updated_at = ?2
             WHERE id = ?1 AND status = 'READY'`)
  .bind(orderId, now)
  .run();

if (claim.meta.changes === 0) {
  // 이미 다른 요청이 선점했거나, 상태가 READY가 아니다
  return existingResultOrConflict(orderId);
}
```

`meta.changes`가 1이면 이 요청이 승인 권한을 선점한 것이고, 0이면 중복 요청이다. D1은 쓰기가 단일 스레드로 직렬화되므로 이 CAS는 확실하게 동작한다.

### 배치로 최종 상태를 원자적으로 반영

카카오페이 `approve` 응답을 받은 뒤, 우리 DB 쪽 변경은 한 번에 묶는다.

```js
const kakao = await approveKakaoPay({ tid, pgToken, orderId, userId });

await env.DB.batch([
  env.DB.prepare(
    `UPDATE orders SET status = 'PAID', updated_at = ?2
      WHERE id = ?1 AND status = 'APPROVING'`
  ).bind(orderId, now),

  env.DB.prepare(
    `UPDATE payments SET aid = ?2, approved_at = ?3 WHERE order_id = ?1`
  ).bind(orderId, kakao.aid, now),

  env.DB.prepare(
    `UPDATE payment_attempts SET result = ?2 WHERE idempotency_key = ?1`
  ).bind(idempotencyKey, JSON.stringify(kakao)),

  env.DB.prepare(
    `INSERT INTO payment_events (order_id, type, payload, created_at)
     VALUES (?1, 'APPROVED', ?2, ?3)`
  ).bind(orderId, JSON.stringify(kakao), now),
]);
```

배치 안의 문장은 서로의 결과를 참조할 수 없다는 점만 주의한다. 필요하면 서브쿼리로 해결한다.

```sql
-- 배치 내에서 다른 테이블 값을 참조해야 할 때
INSERT INTO payment_events (order_id, type, payload, created_at)
SELECT id, 'APPROVED', ?2, ?3 FROM orders WHERE id = ?1 AND status = 'PAID';
```

### 승인 실패 시의 보정

`batch()`가 실패하면 우리 DB는 `APPROVING`에 멈춰 있고 카카오페이는 승인 완료 상태다. 이 불일치는 롤백이 아니라 **재시도와 대사**로 해결한다.

```js
// Cron Trigger로 주기 실행
const stuck = await env.DB.prepare(
  `SELECT o.id, p.tid FROM orders o JOIN payments p ON p.order_id = o.id
    WHERE o.status = 'APPROVING' AND o.updated_at < ?1`
).bind(Date.now() - 5 * 60 * 1000).all();

for (const row of stuck.results) {
  // 카카오페이 주문 조회로 실제 상태 확인
  const actual = await fetch("https://open-api.kakaopay.com/online/v1/payment/order", {
    method: "POST",
    headers: {
      "Authorization": `SECRET_KEY ${env.KAKAO_SECRET_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ cid: env.KAKAO_CID, tid: row.tid }),
  }).then(r => r.json());

  if (actual.status === "SUCCESS_PAYMENT") {
    await reconcileToPaid(env, row.id, actual);   // 앞의 batch 재실행
  } else {
    await markFailed(env, row.id, actual);
  }
}
```

### 더 나은 방법: Cloudflare Workflows로 saga 구성

위의 "상태 저장 + 재시도 + 대사"를 직접 구현하는 대신 Cloudflare Workflows를 쓰면 durable execution을 플랫폼이 대신 해준다. `step.do()`로 감싼 각 단계는 완료 시점이 영속화되고, 실패하면 마지막 성공 단계부터 재개된다.

```js
import { WorkflowEntrypoint } from "cloudflare:workers";

export class KakaoPaymentWorkflow extends WorkflowEntrypoint {
  async run(event, step) {
    const { orderId, userId, amount, itemName } = event.payload;

    const ready = await step.do("kakao-ready", {
      retries: { limit: 3, delay: "2 seconds", backoff: "exponential" },
      timeout: "30 seconds",
    }, async () => {
      return await kakaoReady({ orderId, userId, amount, itemName });
    });

    await step.do("persist-ready", async () => {
      await this.env.DB.batch([
        this.env.DB.prepare(
          "INSERT INTO payments (order_id, tid, amount) VALUES (?1, ?2, ?3)"
        ).bind(orderId, ready.tid, amount),
        this.env.DB.prepare(
          "UPDATE orders SET status = 'READY' WHERE id = ?1 AND status = 'CREATED'"
        ).bind(orderId),
      ]);
    });

    // 사용자 승인 대기 — 리다이렉트 핸들러가 이벤트를 보내면 재개
    const pgToken = await step.waitForEvent("user-approval", {
      type: "pg-token",
      timeout: "15 minutes",
    });

    const approved = await step.do("kakao-approve", {
      retries: { limit: 5, delay: "3 seconds", backoff: "exponential" },
    }, async () => {
      return await kakaoApprove({ tid: ready.tid, pgToken: pgToken.payload.token, orderId, userId });
    });

    await step.do("persist-approved", async () => {
      await this.env.DB.batch([ /* 위의 배치 */ ]);
    });
  }
}
```

주의할 점은 **각 step은 재실행될 수 있으므로 반드시 멱등해야 한다**는 것이다. `kakao-approve` 단계는 같은 `tid` + `pg_token`으로 재호출되면 카카오페이가 중복 승인 에러를 주는데, 이걸 성공으로 간주할지 실패로 볼지 명확히 처리해야 한다. `payment_attempts` 테이블의 캐시된 결과를 먼저 확인하는 방식이 안전하다.

### 강한 직렬화가 필요하면 Durable Object

주문 하나에 대해 진짜 상호 배제가 필요하다면(예: 부분 취소를 여러 번 동시에 요청) Durable Object를 주문 단위로 쓰는 선택지가 있다. DO는 SQLite 스토리지에서 **실제 트랜잭션 API를 제공한다.**

```js
export class OrderActor extends DurableObject {
  async partialCancel(amount) {
    // 이 DO 인스턴스는 단일 스레드로 요청을 직렬 처리한다 (input/output gate)
    return this.ctx.storage.transactionSync(() => {
      const [row] = this.ctx.storage.sql
        .exec("SELECT remaining FROM balance WHERE id = 1").toArray();
      if (row.remaining < amount) throw new Error("insufficient");
      this.ctx.storage.sql.exec(
        "UPDATE balance SET remaining = remaining - ? WHERE id = 1", amount
      );
      return row.remaining - amount;
    });
  }
}
```

정리하면 Cloudflare 안에서 트랜잭션 강도는 다음 순서다.

```text
D1 auto-commit  <  D1 batch()  <  Durable Object transactionSync()
(단일 문장)        (다중 문장 원자성)   (읽고-판단하고-쓰기 원자성)
```

결제 상태 머신 정도는 D1 `batch()` + CAS로 충분하고, 잔액 차감처럼 read-modify-write가 본질인 로직만 DO로 올리는 조합이 현실적이다.

---

## AWS와의 비교

### 트랜잭션 모델

| | Cloudflare D1 | Aurora / RDS (PostgreSQL) | Aurora Serverless v2 | Aurora DSQL | DynamoDB |
|---|---|---|---|---|---|
| 대화형 트랜잭션 | ✗ | ✓ | ✓ | ✓ (낙관적 동시성) | ✗ |
| 다중 문장 원자성 | `batch()` | ✓ | ✓ | ✓ | `TransactWriteItems` (최대 100개) |
| 격리 수준 선택 | ✗ | ✓ | ✓ | 스냅샷 격리 고정 | ✗ |
| 명시적 락 | ✗ | ✓ | ✓ | ✗ (OCC, 충돌 시 재시도) | ✗ (조건부 쓰기로 대체) |
| 커넥션 모델 | HTTP, 풀 불필요 | TCP 커넥션 풀 필요 | 동일 (Proxy 권장) | 커넥션 기반 | HTTP |
| 확장 한계 | DB당 10GB, 단일 스레드 | 인스턴스 크기 | 0~256 ACU 자동 | 사실상 무제한 | 사실상 무제한 |
| 콜드 스타트 | 없음 | 없음(상시 기동) | 0 ACU에서 재개 시 존재 | 거의 없음 | 없음 |

### 서버리스 관점에서 본 실질적 차이

**커넥션 문제가 사라진다.** Lambda + RDS 조합의 고전적 골칫거리는 커넥션 고갈이다. 동시 실행 1000개면 커넥션 1000개를 요구하고, 그래서 RDS Proxy(추가 비용)를 붙인다. D1은 HTTP 기반이라 이 문제가 아예 없다. Workers는 D1 바인딩당 동시 커넥션 6개까지만 쓰고, 그 위는 플랫폼이 알아서 처리한다.

**대신 확장 한계가 다르다.** Aurora는 인스턴스를 키우면 단일 DB로도 상당한 쓰기를 감당한다. D1은 DB당 10GB, 단일 스레드다. Cloudflare 문서도 "하나의 큰 DB보다 여러 개의 작은 DB로 수평 확장하는 데 최적화되어 있다"고 명시한다. 결제량이 커지면 테넌트/기간 단위 샤딩을 애플리케이션에서 직접 설계해야 한다.

**지연 특성이 반대다.**

```text
[AWS]  사용자(서울) → CloudFront/ALB → Lambda(ap-northeast-2) → RDS(같은 VPC)
       엣지까지는 빠르고, 앱-DB 구간은 1ms 미만

[D1]   사용자(서울) → Worker(가장 가까운 엣지, 서울) → D1 primary(위치 힌트에 따름)
       엣지 진입은 매우 빠르지만, 쓰기는 primary까지 왕복해야 한다
```

D1 데이터베이스는 생성 시 위치 힌트를 준다.

```bash
wrangler d1 create payment-db --location=apac
```

한국 사용자를 대상으로 하는데 primary가 북미에 있으면 쓰기 왕복만 150~200ms가 붙는다. 결제처럼 쓰기 위주 워크로드에서는 **위치 힌트 설정이 가장 큰 성능 변수**다. 그리고 `apac`은 리전 힌트일 뿐 "서울"을 보장하지 않는다.

**왕복 횟수 최적화 여지는 D1이 더 크다.** Worker와 D1 사이에 왕복 비용이 있으므로 `batch()`로 묶는 이득이 RDB보다 훨씬 크다. 반대로 말하면 문장 하나씩 `await`으로 실행하는 코드는 D1에서 눈에 띄게 느려진다. 결제 트랜잭션 하나를 배치 1회로 끝내는 설계가 사실상 필수다.

### 비용 비교

D1 요금(2026년 7월 기준, Workers Paid $5/월 포함):

| 항목 | 무료 플랜 | 유료 플랜 포함량 | 초과 단가 |
|---|---|---|---|
| 읽은 행 | 5백만/일 | 250억/월 | $0.001 / 백만 행 |
| 쓴 행 | 10만/일 | 5천만/월 | $1.00 / 백만 행 |
| 스토리지 | 5GB | 5GB | $0.75 / GB·월 |

데이터 전송 비용(egress)은 없다.

**월 10만 건 결제 서비스로 계산해보면** — 결제 1건당 쓴 행 10개(주문 insert, 상태 전이 3회, 결제/시도/이벤트 기록, 인덱스 추가 쓰기), 읽은 행 50개로 잡는다.

```text
쓴 행    100,000 × 10 = 1,000,000행/월   → 포함량(5천만) 안
읽은 행  100,000 × 50 = 5,000,000행/월   → 포함량(250억) 안
스토리지 수백 MB 수준                     → 포함량(5GB) 안
------------------------------------------------------------
D1 비용: $0 (Workers Paid $5/월에 전부 포함)
```

같은 규모를 AWS로 구성하면:

| 구성 | 월 비용(us-east-1 기준 개략) |
|---|---|
| Aurora Serverless v2, 최소 0.5 ACU 상시 | 0.5 × $0.12 × 730h ≈ **$44** + 스토리지 $0.10/GB + I/O $0.20/백만 |
| Aurora Serverless v2, 실사용 1~2 ACU | **$88 ~ $175** |
| RDS PostgreSQL db.t4g.medium Multi-AZ | 약 **$120** |
| Aurora DSQL | $8 / 백만 DPU + $0.33/GB·월, 월 10만 DPU 무료 (사용량 기반이라 소규모는 매우 저렴) |
| DynamoDB On-Demand | 쓰기 $1.25/백만 WRU 수준, 단 `TransactWriteItems`는 **2배 과금** |
| + RDS Proxy, NAT Gateway | 각각 월 $10~40 추가 |

**핵심 차이는 유휴 비용이다.** D1은 쓰지 않으면 스토리지 외에 돈이 들지 않고, 소규모 트래픽은 사실상 $5 안에 다 들어간다. Aurora Serverless v2는 0 ACU까지 축소되지만, 결제 서비스처럼 항상 응답해야 하는 워크로드는 최소 용량을 상시 유지하는 게 보통이라 월 $44부터 시작한다. NAT Gateway나 RDS Proxy 같은 부수 비용도 D1에는 없다.

반대로 **규모가 커지면 우위가 뒤집힐 수 있다.** D1의 "읽은 행"은 반환된 행이 아니라 **스캔한 행** 기준이다. 인덱스 없는 컬럼으로 필터하면 결과가 1건이어도 수만 행을 읽은 것으로 과금된다. 결제 대사 배치처럼 전체 스캔을 도는 작업이 있으면 비용이 빠르게 오른다.

```sql
-- 이 쿼리는 orders 전체를 스캔한다 → 읽은 행 폭증
SELECT * FROM orders WHERE status = 'APPROVING';

-- 인덱스 필수
CREATE INDEX idx_orders_status_updated ON orders(status, updated_at);
```

인덱스는 읽기 비용을 줄이지만, 인덱스 컬럼을 포함한 쓰기는 쓴 행이 추가로 발생한다. 결제 테이블처럼 쓰기가 잦은 곳은 인덱스를 필요한 만큼만 두는 균형이 필요하다.

---

## 판단 기준

D1이 결제에 적합한 경우:

- 트래픽이 중소 규모이고, 유휴 시간 비용을 0에 가깝게 유지하고 싶다
- 이미 Workers 위에 애플리케이션이 올라가 있다 (D1은 같은 런타임 안의 바인딩이라 지연이 최소)
- 결제 상태 머신 수준의 원자성이면 충분하다 (`batch()` + CAS로 커버)
- 인프라 운영 인력을 두고 싶지 않다 (VPC, 커넥션 풀, 프록시, 패치가 전부 없음)

D1을 재고해야 하는 경우:

- **데이터 규모가 10GB를 넘어간다.** 결제 이력은 보존 의무가 있어 계속 쌓인다. 원장은 R2나 별도 분석 스토어로 아카이빙하는 설계를 처음부터 넣어야 한다.
- **복잡한 read-modify-write가 본질인 로직이 많다.** 포인트/적립금 차감, 재고 예약 등. DO로 분리하거나 RDB가 낫다.
- **데이터 소재지 요건이 있다.** D1의 위치 힌트는 리전 수준이고 "국내 보관"을 보장하지 않는다. 국내 전자금융 관련 규제나 개인정보 국외 이전 이슈가 걸린다면 이건 기술 판단이 아니라 법무·컴플라이언스 확인 사항이다. 결제 관련 개인정보를 D1에 직접 넣기 전에 반드시 검토해야 한다.
- **복잡한 정산/집계 쿼리가 필요하다.** 윈도우 함수를 많이 쓰는 리포팅은 스캔 행 과금과 30초 쿼리 제한에 걸린다.

현실적인 절충안은 **결제 트랜잭션 상태만 D1에 두고, 원장과 정산 데이터는 별도 스토어로 분리**하는 구조다.

---

## 정리

- **이해한 것**: D1은 auto-commit이 기본이고, 다중 문장 원자성은 `batch()` 하나로만 제공된다. 대화형 트랜잭션이 없는 것은 네트워크 왕복 중 단일 스레드 DB의 락 점유를 막기 위한 구조적 선택이다. 결제는 애초에 외부 PG 호출을 트랜잭션에 넣을 수 없으므로, 이 제약이 결정적 장애물은 아니다. 조건부 `UPDATE` + `meta.changes` 확인으로 CAS를 만들고, 최종 반영은 `batch()`로 묶고, 불일치는 카카오페이 주문 조회 API로 대사한다.
- **AWS 대비**: 커넥션 풀 문제가 사라지고 유휴 비용이 0에 가까운 대신, DB당 10GB·단일 스레드라는 확장 한계와 스캔 행 기준 과금이라는 다른 종류의 제약이 생긴다. 소규모~중규모에서는 비용 우위가 명확하다(월 $5 vs 월 $44~).
- **아직 확인이 필요한 것**: `apac` 위치 힌트에서 실제 primary가 어느 도시에 배치되는지, 서울에서의 쓰기 왕복 지연 실측치. 결제 관련 개인정보를 D1에 저장할 때의 국외 이전 검토.
- **다음에 볼 것**: Cloudflare Workflows의 `waitForEvent` 타임아웃 처리, Durable Object를 주문 단위로 나눌 때의 인스턴스 수와 비용, D1 read replication을 켰을 때 결제 조회 API의 bookmark 전달 패턴.

## 참고

- [D1 Worker API — D1Database](https://developers.cloudflare.com/d1/worker-api/d1-database/)
- [D1 Limits](https://developers.cloudflare.com/d1/platform/limits/)
- [D1 Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [Durable Objects Storage API](https://developers.cloudflare.com/durable-objects/api/storage-api/)
- [Cloudflare Workflows](https://developers.cloudflare.com/workflows/)
- [카카오페이 온라인 결제 API 포럼](https://developers.kakaopay.com/forum/t/topic/678)
- [Amazon Aurora Pricing](https://aws.amazon.com/rds/aurora/pricing/)
- [Amazon Aurora DSQL Pricing](https://aws.amazon.com/rds/aurora/dsql/pricing/)
