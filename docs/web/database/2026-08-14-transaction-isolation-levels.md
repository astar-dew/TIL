---
title: "트랜잭션 격리 수준: 이상 현상과 DB별 실제 동작"
date: 2026-08-14
tags: [database, transaction, isolation-level, mvcc, lock, deadlock, mysql, postgresql, django]
description: "격리 수준이 막아주는 이상 현상을 정리하고, MySQL과 PostgreSQL의 실제 동작 차이, 재고 차감 같은 갱신 유실을 막는 방법을 정리한다."
---

## 학습 목적

격리 수준은 **동시에 실행되는 트랜잭션이 서로의 작업을 얼마나 볼 수 있는가**를 정하는 설정이다.

평소에는 기본값으로 잘 돌아가다가, 트래픽이 몰리는 순간 재고가 음수가 되거나 포인트가 두 번 차감되는 식으로 터진다. 그리고 이런 문제는 **재현이 어려워서** 원인을 찾는 데 오래 걸린다.

더 곤란한 점은 **표준 정의와 실제 DB 동작이 다르다**는 것이다. "MySQL은 REPEATABLE READ니까 팬텀 리드가 발생한다"는 설명은 교과서적으로는 맞지만 InnoDB의 실제 동작과는 다르다.

이 글에서는 이상 현상을 먼저 정리하고, DB별 실제 동작 차이, 그리고 실무에서 가장 많이 겪는 갱신 유실을 막는 방법을 정리한다.

## 격리성이란

ACID의 I(Isolation)에 해당한다. 이상적으로는 **모든 트랜잭션이 혼자 실행되는 것처럼** 보여야 한다.

문제는 그렇게 만들면 느리다는 것이다. 완전한 격리는 사실상 트랜잭션을 줄 세우는 것과 같다.

```text
완전한 격리  →  안전하지만 느림
느슨한 격리  →  빠르지만 이상 현상 발생
```

격리 수준은 이 사이에서 **어디까지 허용할지 고르는 다이얼**이다.

## 네 가지 이상 현상

### 1. Dirty Read — 커밋되지 않은 값을 읽음

```text
T1: UPDATE stock SET qty = 5      (아직 커밋 안 함)
T2:                                SELECT qty  → 5 를 읽음
T1: ROLLBACK                       (사실 되돌려짐)
T2:                                존재한 적 없는 값으로 로직 수행
```

되돌려질 값을 근거로 결정을 내리게 된다. 대부분의 DB는 기본적으로 이걸 허용하지 않는다.

### 2. Non-Repeatable Read — 같은 행을 두 번 읽었는데 값이 다름

```text
T1: SELECT qty FROM stock WHERE id=1   → 10
T2: UPDATE stock SET qty=3 WHERE id=1; COMMIT
T1: SELECT qty FROM stock WHERE id=1   → 3   (같은 트랜잭션인데 값이 바뀜)
```

트랜잭션 안에서 계산을 나눠 하는 경우 앞뒤가 안 맞는 결과가 나온다.

### 3. Phantom Read — 같은 조건으로 조회했는데 행 수가 다름

```text
T1: SELECT COUNT(*) FROM orders WHERE status='paid'   → 10건
T2: INSERT INTO orders (status) VALUES ('paid'); COMMIT
T1: SELECT COUNT(*) FROM orders WHERE status='paid'   → 11건
```

값이 바뀌는 게 아니라 **없던 행이 나타난다**는 점이 앞의 것과 다르다.

### 4. Lost Update — 갱신 유실

표준 격리 수준 정의에는 없지만 **실무에서 가장 자주 터지는 문제**다.

```text
재고 10개, 두 요청이 동시에 1개씩 차감

T1: SELECT qty → 10
T2: SELECT qty → 10          (둘 다 10을 읽음)
T1: UPDATE qty = 10 - 1 = 9
T2: UPDATE qty = 10 - 1 = 9  (T1의 차감이 사라짐)

결과: 2개를 팔았는데 재고는 1개만 줄었다
```

**읽고 → 계산하고 → 쓰는** 흐름이 애플리케이션 코드에 있으면 격리 수준만으로는 막히지 않는다. 뒤에서 따로 다룬다.

## 표준 격리 수준 4단계

| 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생(표준상) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

이 표가 흔히 외우는 내용인데, **실제 DB 동작은 이보다 복잡하고 대개 더 강하다.**

## DB별 실제 동작

| DB | 기본값 | 실제 특징 |
| --- | --- | --- |
| **MySQL (InnoDB)** | REPEATABLE READ | MVCC 스냅샷 + 넥스트 키 락으로 **팬텀도 대부분 방지** |
| **PostgreSQL** | READ COMMITTED | READ UNCOMMITTED를 요청해도 READ COMMITTED로 동작 |
| **Oracle** | READ COMMITTED | Dirty Read를 아예 지원하지 않음 |
| **SQL Server** | READ COMMITTED (락 기반) | `READ_COMMITTED_SNAPSHOT` 옵션으로 MVCC 방식 전환 |
| **SQLite / D1** | 사실상 직렬화 | 쓰기가 한 번에 하나 |

알아둘 만한 차이 두 가지가 있다.

**MySQL의 REPEATABLE READ는 표준보다 강하다.** 일반 `SELECT`는 트랜잭션 시작 시점의 스냅샷을 보므로 팬텀이 보이지 않고, `SELECT ... FOR UPDATE` 같은 잠금 읽기는 넥스트 키 락으로 범위를 잠가 삽입 자체를 막는다.

**PostgreSQL의 REPEATABLE READ는 스냅샷 격리**다. 팬텀이 발생하지 않는 대신, 충돌이 감지되면 트랜잭션이 실패한다.

```text
ERROR: could not serialize access due to concurrent update
```

즉 PostgreSQL에서 격리 수준을 올리면 **애플리케이션에 재시도 로직이 필요해진다.** 이 차이를 모르고 격리 수준만 올리면 운영에서 에러가 늘어난다.

## MVCC: 읽기가 쓰기를 막지 않는 이유

현대 DB가 빠른 이유의 상당 부분이 MVCC(다중 버전 동시성 제어)에 있다.

행을 덮어쓰는 대신 **버전을 추가로 관리**하고, 각 트랜잭션은 자기 시점에 맞는 버전을 본다.

```text
행 id=1

버전 A (qty=10)  ← T1이 보는 버전 (T1 시작 시점의 스냅샷)
버전 B (qty=3)   ← T2가 방금 만든 최신 버전
```

여기서 나오는 결론이 중요하다.

- **읽기는 쓰기를 기다리지 않는다.** 조회 때문에 갱신이 막히지 않는다.
- **그래서 일반 SELECT는 락을 걸지 않는다.** 재고 확인을 `SELECT`로 했다고 안전해지지 않는다.
- 오래된 버전은 정리되어야 한다. PostgreSQL의 `VACUUM`, MySQL의 언두 로그 정리가 그 작업이다.

**긴 트랜잭션이 위험한 이유**도 여기 있다. 트랜잭션이 열려 있는 동안 그 시점의 버전을 계속 보존해야 해서 저장 공간과 성능이 나빠진다.

## 갱신 유실을 막는 세 가지 방법

실무에서 가장 자주 필요한 부분이다. 재고 차감을 예로 든다.

### 1. 원자적 UPDATE — 가장 단순하고 빠르다

읽지 말고 **DB에서 계산하게** 한다.

```sql
UPDATE stock
   SET qty = qty - 1
 WHERE id = 1 AND qty >= 1;
```

영향받은 행이 0이면 재고가 부족했던 것이다.

```python
# Django
updated = Stock.objects.filter(id=1, qty__gte=1).update(qty=F("qty") - 1)
if updated == 0:
    raise OutOfStock()
```

`F()` 표현식은 Python에서 계산하지 않고 SQL로 내려보내므로 읽기-쓰기 간극이 없다. **가능하면 이 방법을 쓴다.**

### 2. 비관적 락 — 먼저 잠그고 작업

충돌이 자주 일어난다고 가정하고, 읽을 때부터 잠근다.

```sql
BEGIN;
SELECT qty FROM stock WHERE id = 1 FOR UPDATE;   -- 다른 트랜잭션은 여기서 대기
-- 복잡한 검증 로직
UPDATE stock SET qty = qty - 1 WHERE id = 1;
COMMIT;
```

```python
with transaction.atomic():
    stock = Stock.objects.select_for_update().get(id=1)
    if stock.qty < 1:
        raise OutOfStock()
    stock.qty -= 1
    stock.save()
```

| 언제 쓰나 | 주의점 |
| --- | --- |
| 계산이 복잡해 SQL 한 줄로 안 될 때 | 대기가 길어지면 처리량이 떨어진다 |
| 여러 테이블을 함께 확인해야 할 때 | 락 순서가 엇갈리면 데드락이 난다 |
| 충돌이 실제로 잦을 때 | 트랜잭션을 짧게 유지해야 한다 |

`FOR UPDATE`는 쓰기 잠금, `FOR SHARE`는 읽기 잠금이다. `NOWAIT`나 `SKIP LOCKED`를 붙이면 대기 대신 즉시 실패하거나 잠긴 행을 건너뛴다. 작업 큐를 DB로 구현할 때 `SKIP LOCKED`가 유용하다.

### 3. 낙관적 락 — 버전으로 충돌 감지

충돌이 드물다고 가정하고, 잠그지 않되 **바뀌었으면 실패**시킨다.

```sql
UPDATE stock
   SET qty = 9, version = version + 1
 WHERE id = 1 AND version = 3;
```

영향받은 행이 0이면 그사이 누군가 바꾼 것이므로 다시 읽고 재시도한다.

| 방식 | 적합한 상황 |
| --- | --- |
| 원자적 UPDATE | 단순 증감. **1순위로 검토** |
| 비관적 락 | 충돌이 잦고 로직이 복잡함 |
| 낙관적 락 | 충돌이 드물고 처리량이 중요함. 사용자 편집 화면 등 |

## 데드락

서로가 가진 락을 기다리면 아무도 진행하지 못한다.

```text
T1: A 잠금 → B 요청 (대기)
T2: B 잠금 → A 요청 (대기)
```

MySQL과 PostgreSQL은 데드락을 감지해 한쪽을 강제 롤백한다. 애플리케이션은 이 에러를 받아 **재시도**해야 한다.

예방책은 두 가지다.

1. **락 획득 순서를 통일한다.** 항상 ID 오름차순으로 잠그는 식이면 순환이 생기지 않는다.
2. **트랜잭션을 짧게 유지한다.** 락을 쥔 시간이 짧을수록 충돌 확률이 낮다.

특히 트랜잭션 안에서 **외부 API 호출을 하지 않는다.** 응답이 늦어지면 그동안 락이 유지된다.

```python
# 나쁨 — 결제 API 응답 동안 락이 유지된다
with transaction.atomic():
    order = Order.objects.select_for_update().get(id=1)
    result = payment_api.charge(order.amount)   # 수 초가 걸릴 수 있다
    order.status = "paid"
    order.save()
```

## 실무 선택 기준

### 기본값을 그대로 두는 편이 낫다

격리 수준을 전역으로 올리면 **모든 쿼리가 비용을 치른다.** 대부분의 서비스는 기본값을 유지하고, 문제가 되는 지점에만 락이나 원자적 UPDATE를 적용하는 편이 낫다.

```text
전역 격리 수준 ↑         →  전체 성능 저하, 재시도 증가
문제 쿼리에만 락 적용    →  필요한 곳만 비용 지불   ← 권장
```

### 트랜잭션은 짧게

격리 수준을 올리는 것보다 **트랜잭션 범위를 줄이는 것**이 효과가 클 때가 많다.

- 조회만 하는 작업을 트랜잭션 안에 넣지 않는다
- 외부 API 호출, 파일 업로드, 이메일 발송을 트랜잭션 밖으로 뺀다
- 사용자 입력을 기다리는 동안 트랜잭션을 열어두지 않는다

### 어떤 작업에 무엇이 필요한가

| 작업 | 권장 |
| --- | --- |
| 조회 전용 API | 기본값. 락 불필요 |
| 조회수·좋아요 증가 | 원자적 UPDATE |
| 재고 차감, 잔액 차감 | 원자적 UPDATE + 조건, 또는 비관적 락 |
| 결제 상태 전이 | 조건부 UPDATE로 상태 머신 구성 |
| 정산·집계 배치 | 일관된 스냅샷이 필요하면 격리 수준 상향 |
| 사용자 편집 폼 | 낙관적 락 (버전 컬럼) |

## 서버리스와 분산 환경에서는 다르다

이 글의 내용은 단일 DB 인스턴스를 전제한다. 환경이 달라지면 쓸 수 있는 수단도 달라진다.

- **읽기 복제본**을 쓰면 복제 지연 때문에 방금 쓴 값이 안 보일 수 있다. 쓰기 직후 조회는 주 DB로 보내야 한다.
- **서버리스 DB**는 대화형 트랜잭션을 제한하는 경우가 많다. Cloudflare D1처럼 `SELECT FOR UPDATE`를 쓸 수 없는 환경에서는 **조건부 UPDATE 기반 상태 머신**으로 풀어야 한다. 이 방식은 [서버리스 DB의 트랜잭션 처리](./2026-07-30-serverless-transaction-cloudflare-d1.md)에 정리했다.
- **분산 트랜잭션**이 필요해지면 격리 수준이 아니라 사가(Saga)나 멱등성 설계의 영역이 된다.

## 자주 하는 실수

### `SELECT`로 확인하고 `UPDATE` 하기

MVCC에서 일반 `SELECT`는 락을 걸지 않는다. 확인과 갱신 사이에 다른 트랜잭션이 끼어든다.

### 격리 수준만 올리면 안전해진다고 생각하기

갱신 유실은 애플리케이션의 읽기-쓰기 흐름에서 생긴다. PostgreSQL에서 수준을 올리면 에러가 나고, 재시도 로직이 없으면 실패로 이어진다.

### 트랜잭션 안에서 외부 API 호출

응답을 기다리는 동안 락이 유지되어 다른 요청이 밀린다.

### 데드락 재시도 로직 없음

데드락은 정상적으로 발생할 수 있는 상황이다. 감지되면 재시도해야 한다.

### DB를 바꿨는데 동작이 같을 거라 가정하기

MySQL에서 문제없던 코드가 PostgreSQL에서 직렬화 오류를 낼 수 있다. 기본값과 동작이 다르다.

### 필요 없는 곳까지 트랜잭션으로 감싸기

단순 조회를 트랜잭션에 넣으면 스냅샷이 유지되어 정리 작업이 밀린다.

## 정리

- 격리 수준은 동시 실행 트랜잭션이 서로를 얼마나 볼지 정하는 설정이며, 안전성과 성능의 트레이드오프다.
- 이상 현상은 Dirty Read, Non-Repeatable Read, Phantom Read이고, 여기에 표준에는 없지만 실무에서 가장 잦은 갱신 유실이 있다.
- 표준 표만 외우면 실제 동작을 놓친다. MySQL의 REPEATABLE READ는 팬텀도 대부분 막고, PostgreSQL은 수준을 올리면 직렬화 오류와 재시도가 필요해진다.
- MVCC 덕분에 읽기는 쓰기를 막지 않지만, 그래서 일반 `SELECT`는 안전장치가 되지 못한다.
- 갱신 유실은 원자적 UPDATE를 1순위로, 로직이 복잡하면 비관적 락, 충돌이 드물면 낙관적 락으로 푼다.
- 데드락은 락 순서 통일과 짧은 트랜잭션으로 예방하고, 발생 시 재시도한다.
- 전역 격리 수준을 올리기보다 문제 지점에만 락을 적용하는 편이 낫다.
- 트랜잭션 안에서 외부 API를 호출하지 않는다.

## 학습 체크리스트

- [ ] 네 가지 이상 현상을 타임라인으로 설명할 수 있는가?
- [ ] Non-Repeatable Read와 Phantom Read의 차이를 구분할 수 있는가?
- [ ] 담당 서비스 DB의 기본 격리 수준을 알고 있는가?
- [ ] MySQL과 PostgreSQL의 REPEATABLE READ 동작 차이를 설명할 수 있는가?
- [ ] MVCC에서 일반 `SELECT`가 락을 걸지 않는다는 점과 그 함의를 아는가?
- [ ] 재고 차감 코드가 갱신 유실에 안전한지 확인해 봤는가?
- [ ] 원자적 UPDATE, 비관적 락, 낙관적 락을 상황에 맞게 고를 수 있는가?
- [ ] 데드락 발생 시 재시도 로직이 있는가?
- [ ] 트랜잭션 안에서 외부 API를 호출하는 코드가 없는지 확인했는가?

## 참고

- [MySQL — Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL — InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Django — Database transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/)
- [Django — `select_for_update()`](https://docs.djangoproject.com/en/stable/ref/models/querysets/#select-for-update)
- [Martin Kleppmann — Designing Data-Intensive Applications, 7장](https://dataintensive.net/)
