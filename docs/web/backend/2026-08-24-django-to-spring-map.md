---
title: "Django 개발자를 위한 Spring 지도: 무엇이 같고 무엇이 진짜 다른가"
date: 2026-08-24
tags: [spring, java, django, backend, career]
description: "Django/DRF 경험을 Spring 용어로 번역하고, 채용 공고의 기술 스택을 해독하고, 정말로 새로 배워야 할 세 가지를 가려낸다."
---

## 학습 목적

채용 공고에 Spring이 압도적으로 많다. Django/DRF로 실무를 해왔더라도 공고를 열면 `JPA`, `QueryDSL`, `Spring Batch`, `MSA` 같은 단어가 줄줄이 나와서 **지원 여부를 판단하는 것부터 막힌다.**

그런데 문제는 대개 "Spring을 못 짜서"가 아니다. 두 가지가 진짜 장벽이다.

| 장벽 | 증상 |
| --- | --- |
| 공고 해독 불가 | 무슨 말인지 몰라 지원 가능 여부 자체를 판단 못 한다 |
| 경험 번역 불가 | Django로 한 일을 Spring 언어로 설명 못 해 경력이 0으로 취급된다 |

두 번째 손실이 훨씬 크다. **Django에서 해온 일의 상당수는 Spring에도 같은 개념으로 존재하는데, 이름이 달라서 어필되지 않을 뿐이다.**

이 글의 목표는 하나다 — **공고를 읽고 "이건 내가 아는 것", "이건 이름만 다른 것", "이건 진짜 모르는 것"으로 분류할 수 있는 상태.** Spring 코드를 잘 짜는 법은 다루지 않는다. 지도를 먼저 그리고, 각 지점은 후속 글에서 판다.

## 먼저 알아야 할 두 프레임워크의 성격 차이

Django와 Spring은 **개발자에게 요구하는 것이 반대**다.

```text
Django          "이 자리에 이 파일을 놓아라"
                models.py, views.py, urls.py, admin.py
                → 자리가 정해져 있다. 빨리 시작할 수 있다.

Spring          "무엇을 쓸지 골라서 조립해라"
                스타터를 고르고, 빈을 등록하고, 설정한다
                → 자유롭다. 그래서 처음엔 더 어렵다.
```

Django에 익숙한 사람이 Spring을 처음 볼 때 당황하는 지점이 대개 여기다. **"그래서 파일을 어디에 만들어야 하죠?"** 에 대한 정답이 없다. `controller / service / repository` 같은 관례는 있지만 프레임워크가 강제하지 않는다.

### 애노테이션은 데코레이터가 아니다

이게 이 글에서 가장 중요한 한 가지다. 파이썬 데코레이터와 자바 애노테이션은 생긴 게 비슷해서 같은 것으로 착각하기 쉬운데, **작동 원리가 근본적으로 다르다.**

```python
# Python: 데코레이터는 함수를 실제로 교체한다
@transaction.atomic
def save(self):
    ...
# save는 이미 "감싸진 함수"다. 누가 부르든 트랜잭션이 걸린다.
```

```java
// Java: 애노테이션은 그냥 메타데이터다. 아무 동작도 하지 않는다
@Transactional
public void save() {
    ...
}
// save는 여전히 원본 메서드다. 누군가 이 표시를 읽고 처리해 줘야 한다.
```

애노테이션은 **"나 이런 표시가 붙어 있어요"라는 쪽지**일 뿐이다. 실제로 트랜잭션을 걸어 주는 건 Spring이 런타임에 만든 **프록시 객체**다. 이 한 가지 차이에서 Spring 초보자가 겪는 사고의 절반이 나온다. 뒤에서 다시 다룬다.

## 거의 그대로 옮겨지는 것들

먼저 안심할 부분이다. 아래는 이름만 익히면 되는 것들이다.

| Django / DRF | Spring | 비고 |
| --- | --- | --- |
| `manage.py runserver` | 내장 Tomcat + `main()` | Boot가 서버를 품고 있다 |
| `settings.py` | `application.yml` + `@ConfigurationProperties` | 타입 있는 설정 객체로 받는다 |
| `requirements.txt` / pip | `build.gradle` / Gradle·Maven | 의존성 선언 |
| `urls.py` | `@GetMapping` 등 | **한곳에 모으지 않고 컨트롤러에 분산** |
| `views.py` | `@RestController` | |
| `models.py` | `@Entity` | 매핑 방식은 다르다 (뒤에서) |
| DRF `Serializer` | DTO(주로 `record`) + Bean Validation | 직렬화와 검증이 분리된다 |
| `transaction.atomic` | `@Transactional` | 개념은 거의 같다 |
| `migrations` | Flyway / Liquibase | SQL을 직접 쓴다 |
| Celery | `@Async` / Spring Batch / Quartz | 용도별로 나뉜다 |
| pytest | JUnit 5 + `@SpringBootTest` / `MockMvc` | |
| django-debug-toolbar | p6spy / Hibernate SQL 로그 | 쿼리 확인 |

`urls.py`가 없다는 점이 처음엔 불편하다. 전체 API 목록을 한눈에 보려면 Swagger(springdoc-openapi)를 붙이거나 IDE의 엔드포인트 목록 기능을 쓴다.

**Django Admin에 대응하는 것은 없다.** 이건 Django의 강력한 차별점이고, Spring 진영에서는 관리 화면을 직접 만들거나 별도 도구를 붙인다. 공고에서 "어드민 개발" 이야기가 나오면 이 맥락이다.

## 이름은 비슷한데 실제로는 다른 것들

여기서부터 주의해야 한다.

### Middleware → Filter와 Interceptor, 두 계층

Django의 Middleware 하나가 Spring에서는 **두 개의 층으로 나뉜다.**

```text
요청
 │
 ├─ Filter          서블릿 레벨. Spring 바깥이다.
 │                  인증(Spring Security), CORS, 인코딩, 요청 로깅
 │
 ├─ DispatcherServlet
 │
 ├─ Interceptor     Spring MVC 안. 어떤 핸들러가 처리할지 안다.
 │                  권한 세부 체크, 실행 시간 측정
 │
 └─ Controller
```

차이가 실무에서 갈리는 지점은 이렇다. **Filter는 Spring 빈과 예외 처리 체계 바깥**이라 `@ExceptionHandler`가 잡아 주지 않는다. Filter에서 던진 예외는 따로 처리해야 한다. 반면 Interceptor는 어떤 핸들러가 처리할지 알고 있어서 실행 시간 측정이나 로깅 같은 부가 관심사에 쓰기 좋다.

주의할 것이 하나 있다. **Spring 공식 문서는 Interceptor를 보안 계층으로 쓰는 것을 권장하지 않는다.** 인터셉터의 경로 패턴과 컨트롤러 매핑이 어긋날 수 있고 그 틈이 인증 우회가 되기 때문이다. 문서는 Spring Security나 서블릿 필터 체인을 가능한 한 이른 시점에 적용하라고 안내한다.

Django Middleware 이야기는 [Django Middleware와 Signal의 동작 방식](./2026-07-10-django-middleware-signal.md)에 정리해 뒀다.

### Signal → ApplicationEvent, 그런데 트랜잭션 문제가 풀린다

Django Signal의 고질적인 문제가 있다. **트랜잭션이 롤백돼도 Signal은 이미 실행돼 버린다.** 주문 저장에 실패했는데 알림 메일은 나가는 식이다.

Spring에는 이걸 위한 장치가 있다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    // 커밋이 실제로 끝난 뒤에만 실행된다
    notificationService.send(event.orderId());
}
```

Django에서 `transaction.on_commit()`으로 손수 처리하던 것이 프레임워크 기능으로 들어와 있다. **면접에서 Signal 경험을 이야기할 때 이 대응 관계를 알고 있으면 좋다.** "Django Signal을 쓰다가 롤백 시 부작용 문제를 겪었고, Spring에서는 `@TransactionalEventListener`가 그 지점을 다룬다"는 식의 설명은 실무 경험이 있다는 신호다.

기본이 **동기 실행**이라는 점은 양쪽 다 같다. 비동기로 하려면 `@Async`를 붙여야 한다.

### ORM → 여기가 진짜 다르다

`models.py` → `@Entity`로 옮기면 될 것 같지만, **동작 모델이 완전히 다르다.** 절을 따로 뺀다.

## 진짜 새로 배워야 하는 것 ①: DI 컨테이너

Django에는 대응물이 아예 없다. Django에서는 그냥 import해서 쓴다.

```python
from myapp.services import EmailService

def create_order(...):
    EmailService().send(...)      # 직접 만들어 쓴다
```

Spring에서는 **객체를 내가 만들지 않는다.** 컨테이너가 만들어서 넣어 준다.

```java
@Service
public class OrderService {

    private final EmailSender emailSender;

    // 생성자로 받는다. Spring이 알아서 넣어 준다
    public OrderService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

### 왜 이런 게 필요했나

**동적 언어에서는 덜 필요했기 때문이다.** 파이썬은 테스트할 때 `mock.patch`로 런타임에 무엇이든 바꿔치기할 수 있다. 굳이 주입 지점을 설계하지 않아도 된다.

자바는 그게 안 된다. 그래서 **바꿔 끼울 자리를 미리 만들어 두는 방식**이 발달했고, 그 자리를 관리하는 게 컨테이너다. 얻는 것과 잃는 것이 분명하다.

| 얻는 것 | 잃는 것 |
| --- | --- |
| 테스트에서 가짜 구현으로 교체 쉬움 | 코드를 읽을 때 실제 구현을 바로 못 찾음 |
| 구현체 교체 시 사용처 수정 불필요 | 설정과 추상화 계층이 늘어남 |
| 객체 생명주기를 컨테이너가 관리 | 시작 시점에 무슨 일이 벌어지는지 불투명 |

### 최소한 알아야 할 두 가지

**생성자 주입을 쓴다.** `@Autowired` 필드 주입은 안티패턴으로 본다. 필드에 직접 넣으면 `final`을 못 붙이고, 테스트에서 리플렉션 없이는 값을 넣을 수 없으며, 순환 참조가 있어도 실행될 때까지 모른다.

**빈은 기본이 싱글톤이다.** 애플리케이션 전체에서 하나만 만들어져 공유된다. 따라서 **빈에 상태를 두면 스레드 안전성 사고가 난다.** Django의 뷰 함수는 매 요청 호출되고 상태를 안 갖는 게 자연스러워서 이 감각이 없을 수 있다.

## 진짜 새로 배워야 하는 것 ②: 영속성 컨텍스트

Django 개발자가 JPA에서 가장 크게 깨지는 지점이다. 세 가지가 낯설다.

### "저장을 안 했는데 저장된다"

```python
# Django: save()를 불러야 UPDATE가 나간다
order = Order.objects.get(id=1)
order.status = "PAID"
order.save()                       # ← 이때 UPDATE
```

```java
// JPA: save()를 부르지 않아도 UPDATE가 나간다
@Transactional
public void pay(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    order.setStatus(Status.PAID);
    // save() 없음. 트랜잭션이 끝날 때 자동으로 UPDATE가 나간다
}
```

조회한 엔티티는 **영속 상태**가 되어 영속성 컨텍스트가 감시한다. 트랜잭션이 끝날 때 처음 값과 비교해서 바뀐 필드만 UPDATE를 만든다. 이걸 **더티 체킹**이라 한다.

### "조회했는데 쿼리가 안 나간다"

같은 트랜잭션 안에서 같은 ID를 두 번 조회하면 두 번째는 쿼리가 나가지 않는다. 영속성 컨텍스트가 **1차 캐시** 역할을 하기 때문이다. 그래서 SQL 로그만 보고 "조회가 누락됐나" 오해하기 쉽다.

### "트랜잭션 밖에서 접근하면 터진다"

```text
LazyInitializationException: could not initialize proxy - no Session
```

지연 로딩으로 설정된 연관 엔티티는 실제 접근 시점에 쿼리를 날리는데, 트랜잭션이 이미 끝났으면 날릴 수가 없다. **Django에는 이런 제약이 없다.** QuerySet은 어디서든 평가되면 쿼리가 나간다.

그래서 JPA에서는 "필요한 데이터를 트랜잭션 안에서 미리 확보한다"는 사고가 필요하다.

### N+1은 양쪽 다 있다

다행히 이건 이미 아는 문제다. 대응 관계만 알면 된다.

| Django | JPA |
| --- | --- |
| `select_related()` | fetch join, `@EntityGraph` |
| `prefetch_related()` | `@BatchSize`, `hibernate.default_batch_fetch_size` |
| `django-debug-toolbar`로 쿼리 확인 | SQL 로그 / p6spy |

N+1 자체의 원리는 [Django ORM과 DRF의 N+1 문제 최적화](./2026-07-20-django-drf-internals-and-tradeoffs.md)에 정리해 뒀다. **이 경험은 그대로 자산이다.** 면접에서 "N+1을 겪고 쿼리 수를 측정해서 잡아봤다"는 이야기는 ORM 종류와 무관하게 통한다.

## 진짜 새로 배워야 하는 것 ③: AOP 프록시

앞서 말한 "애노테이션은 데코레이터가 아니다"가 여기서 사고로 나타난다. **Spring에서 가장 유명한 함정**이다.

```java
@Service
public class OrderService {

    public void processAll(List<Long> ids) {
        for (Long id : ids) {
            this.saveOne(id);        // ← 트랜잭션이 걸리지 않는다
        }
    }

    @Transactional
    public void saveOne(Long id) {
        ...
    }
}
```

`saveOne`에 `@Transactional`이 분명히 붙어 있는데 트랜잭션이 안 걸린다. 이유는 이렇다.

```text
외부에서 호출할 때
  호출자 ──▶ [프록시] ──▶ 실제 OrderService
             트랜잭션 시작/커밋을 여기서 한다

내부에서 this로 호출할 때
  processAll ──▶ saveOne          ← 프록시를 안 거친다
                                     애노테이션은 그냥 표시일 뿐
```

Spring이 주입하는 것은 `OrderService` 원본이 아니라 **그것을 감싼 프록시**다. 프록시를 통과할 때만 트랜잭션이 시작된다. `this.saveOne()`은 원본 객체 안에서 원본 메서드를 부르는 것이라 프록시를 지나가지 않는다.

파이썬 데코레이터였다면 이런 일이 없다. 데코레이터는 함수 객체 자체를 교체하므로 `self.save_one()`으로 불러도 적용된다. **이 차이를 모르면 "붙였는데 왜 안 되지"에서 몇 시간을 버린다.**

해결은 클래스를 분리하거나(가장 흔함), 자기 자신을 주입받거나, `TransactionTemplate`을 쓰는 것이다.

이 프록시 구조는 `@Transactional`뿐 아니라 `@Cacheable`, `@Async`, `@PreAuthorize` 등 애노테이션 기반 기능 전부에 똑같이 적용된다. **하나를 이해하면 전부 이해된다.**

## 채용 공고 용어 해독

공고에서 자주 보이는 단어들이다. "없으면 지원 못 하나" 열이 실제 판단 기준이다.

| 용어 | 뜻 | Django 쪽 대응 | 없으면 지원 못 하나 |
| --- | --- | --- | --- |
| Spring Boot | 설정을 자동화한 Spring 배포판 | Django 그 자체 | 사실상 전제. 필수 |
| JPA / Hibernate | 자바 표준 ORM | Django ORM | **필수급.** 여기에 시간을 쓴다 |
| QueryDSL | 타입 안전한 동적 쿼리 빌더 | `Q` 객체 조합 | 있으면 좋음. 빨리 익힌다 |
| MyBatis | SQL을 직접 쓰는 매퍼 | raw SQL | 금융·공공·레거시에 많다 |
| Spring Security | 인증·인가 프레임워크 | `django.contrib.auth` | 개념만 알아도 지원 가능 |
| Spring Batch | 대용량 배치 처리 | Celery + 관리 명령 | 배치 공고가 아니면 선택 |
| MSA | 서비스 분리 아키텍처 | — | 경험 없어도 지원 가능 |
| Kafka / RabbitMQ | 메시지 브로커 | Celery 브로커 | 개념 이해로 대응 |
| Gradle / Maven | 빌드 도구 | pip + setup.py | 반나절이면 충분 |
| JUnit / Mockito | 테스트 프레임워크 | pytest / unittest.mock | 개념 동일 |
| WebFlux | 논블로킹 리액티브 | async Django | 소수. 없어도 무방 |

정리하면 **실제로 벽이 되는 건 JPA 하나**다. 나머지는 개념 대응으로 상당 부분 넘어가거나 입사 후 익혀도 되는 것들이다. 공고의 스택 목록에 겁먹을 필요가 없다는 뜻이다.

### 버전은 어디쯤 와 있나

2026년 8월 기준으로 Spring Boot는 4.1.x, Spring Framework는 7.0.x가 최신이다. Boot 4는 Spring Framework 7, Jakarta EE 10, Jackson 3을 요구하며 Java 21 이상이 기준이다.

다만 **공고에 적힌 버전은 대개 이보다 낮다.** Boot 2.7이나 3.x가 흔하고, `javax` → `jakarta` 패키지 전환(Boot 3.0에서 일어났다)을 아직 안 한 회사도 많다. 학습은 최신으로 하되, **공고에 낮은 버전이 적혀 있다고 이상하게 볼 필요는 없다.**

## 그래서 무엇부터 할 것인가

순서가 중요하다. 위에서부터 한다.

```text
1. 자바 문법 최소한          record, Optional, 스트림, 제네릭
                             (전부 배우려 하지 않는다)
2. Spring Boot로 CRUD 하나   컨트롤러-서비스-리포지토리 감각
3. JPA를 제대로              영속성 컨텍스트, 더티 체킹, N+1
                             ← 여기에 시간의 절반을 쓴다
4. @Transactional과 프록시   왜 안 걸리는지 설명할 수 있게
5. 나머지는 공고 보고 필요한 것만
```

3번에 시간을 몰아야 한다. **JPA를 모르면 코드를 읽을 수조차 없고, 알면 나머지는 따라온다.**

### 지원할 때의 전략

`career/` 문서의 기준으로 보면, 지금 필요한 건 **"Spring을 안다"가 아니라 "증거"**다. 가장 효율이 좋은 건 이것이다.

> **이미 Django로 만든 것 중 하나를 Spring으로 이식하고, 그 과정을 글로 남긴다.**

이러면 세 가지가 한 번에 나온다.

- Spring 코드라는 결과물
- **두 프레임워크를 비교할 수 있다는 신호** — 신입이 아니라 경력자만 할 수 있는 이야기다
- 면접에서 쓸 구체적 소재 ("Django ORM에서는 이렇게 했는데 JPA에서는 이 지점이 달랐다")

새 토이 프로젝트를 처음부터 만드는 것보다 낫다. **비교 대상이 있는 결과물**이기 때문이다.

## 자주 하는 오해

**"자바를 다 배우고 Spring을 시작해야 한다"** — 아니다. 실무 Spring 코드에 쓰이는 자바 문법은 생각보다 좁다. 문법 책을 끝내려다 지치는 경우가 훨씬 많다.

**"Django 경력은 인정 안 될 것이다"** — 프레임워크 문법은 대체 가능하고, **ORM으로 성능 문제를 겪고 해결한 경험, 트랜잭션 경계를 설계한 경험은 그대로 인정된다.** 문제는 그것을 Spring 용어로 번역해서 말하느냐다.

**"`@Transactional`을 붙이면 트랜잭션이 걸린다"** — 프록시를 거치는 호출일 때만이다. 이걸 아는 것과 모르는 것이 면접에서 갈린다.

**"JPA는 SQL을 몰라도 된다는 뜻이다"** — 반대다. 어떤 SQL이 나가는지 모르면 N+1과 의도치 않은 UPDATE에 그대로 당한다. SQL을 더 알아야 한다.

## 정리

- 막히는 지점은 Spring 문법이 아니라 **공고 해독과 경험 번역**이다.
- Django에서 하던 일의 상당수는 이름만 다르다. 설정, 라우팅, 뷰, 직렬화, 트랜잭션, 이벤트 모두 대응물이 있다.
- **애노테이션은 데코레이터가 아니다.** 그 자체로는 아무 동작도 안 하고, 프록시가 읽어서 처리한다. Spring 사고의 절반이 여기서 나온다.
- 진짜 새로 배울 것은 세 가지다 — **DI 컨테이너, 영속성 컨텍스트, AOP 프록시.**
- 이 중 **JPA(영속성 컨텍스트)에 학습 시간의 절반을 쓴다.** 나머지는 개념 대응으로 넘어간다.
- 공고의 긴 스택 목록에서 실제로 벽인 것은 JPA 정도다. MSA, Kafka, Batch는 경험 없이도 지원 가능하다.
- 가장 효율 좋은 증거는 **기존 Django 결과물을 Spring으로 이식하고 비교 글을 쓰는 것**이다.

## 학습 체크리스트

- [ ] 파이썬 데코레이터와 자바 애노테이션의 차이를 설명할 수 있는가
- [ ] Django Middleware가 Spring에서 두 계층으로 나뉘는 이유와, 각각을 언제 쓰는지 아는가
- [ ] Django Signal의 롤백 문제와 `@TransactionalEventListener`의 관계를 설명할 수 있는가
- [ ] Django에는 DI 컨테이너가 없어도 됐던 이유를 언어 특성으로 설명할 수 있는가
- [ ] 빈이 싱글톤이라는 사실이 왜 스레드 안전성 문제로 이어지는지 아는가
- [ ] `save()`를 부르지 않았는데 UPDATE가 나가는 이유를 설명할 수 있는가
- [ ] `LazyInitializationException`이 언제 발생하는지, Django에는 왜 이 개념이 없는지 아는가
- [ ] `this.method()` 호출에서 `@Transactional`이 무시되는 이유를 그림으로 그릴 수 있는가
- [ ] 공고의 스택 목록을 보고 "필수 / 있으면 좋음 / 없어도 됨"으로 분류할 수 있는가

## 참고

- [Spring Framework Documentation](https://docs.spring.io/spring-framework/reference/)
- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/index.html)
- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)
- [Transaction Management — Spring Docs](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Django documentation](https://docs.djangoproject.com/en/stable/)
