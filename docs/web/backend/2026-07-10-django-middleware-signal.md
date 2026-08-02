---
title: "Django Middleware와 Signal의 동작 방식 및 트레이드오프"
date: 2026-07-10
tags: [django, middleware, signal, transaction, architecture]
description: "Django 요청 처리 과정에서 Middleware와 Signal이 동작하는 방식과 사용 시 발생하는 실행 순서, 트랜잭션, 유지보수 트레이드오프를 정리한다."
---

## 학습 목적

Django Middleware와 Signal은 여러 기능에서 재사용할 수 있는 강력한 확장 지점이다. 하지만 실행 순서와 호출 시점을 모르면 인증·로깅·예외 처리의 동작이 달라지거나, 모델 저장만으로 예상하지 못한 부가 작업이 실행될 수 있다.

이 글에서는 Django 요청이 Middleware와 view를 통과하는 흐름을 먼저 살펴보고, Signal의 동기 실행 방식과 남용 시 발생하는 문제를 정리한다.

## Django 요청 처리 흐름

Django 요청은 대략 다음 순서로 처리된다.

```text
웹 서버
  -> WSGI/ASGI handler
  -> Middleware 요청 방향
  -> URL resolver
  -> View 또는 DRF View
  -> ORM/서비스 로직
  -> Response 생성
  -> Middleware 응답 방향
  -> 웹 서버
```

DRF를 사용하면 Django view 내부에 API 처리 단계가 추가된다.

```text
HttpRequest
  -> DRF Request 변환
  -> 인증
  -> 권한
  -> throttling
  -> handler/list/retrieve
  -> Serializer
  -> Renderer
  -> Response
```

따라서 모든 요청에 적용할 공통 관심사는 Middleware에, 특정 API의 업무 규칙은 view·permission·serializer·service 계층에 두는 것이 기본적인 경계가 된다.

## Middleware란

Middleware는 Django의 요청과 응답 사이에 끼워 넣는 전역 처리 계층이다. `settings.py`의 `MIDDLEWARE` 목록에 등록하면 각 요청을 처리할 때 실행된다.

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "project.middleware.RequestLoggingMiddleware",
]
```

각 Middleware는 다음 계층을 나타내는 `get_response`를 받아 요청을 전달하고, 반환된 응답을 다시 가공한다.

```python
class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # 요청 방향 처리
        response = self.get_response(request)
        # 응답 방향 처리
        return response
```

`__init__()`은 서버 시작 시 한 번 호출되고, `__call__()`은 요청마다 호출된다. 요청마다 변하는 상태를 Middleware 인스턴스의 속성에 저장하면 동시 요청 사이에 값이 섞일 수 있으므로 주의해야 한다.

## Middleware 처리 순서

요청은 `MIDDLEWARE`에 적힌 순서대로 바깥에서 안쪽으로 들어가고, view가 반환한 응답은 역순으로 바깥으로 나온다.

```text
요청
Security
  -> Session
    -> Authentication
      -> RequestLogging
        -> View
      <- RequestLogging
    <- Authentication
  <- Session
<- Security
응답
```

이를 onion 구조라고 생각할 수 있다. `AuthenticationMiddleware`가 session을 사용해 `request.user`를 만들기 때문에 `SessionMiddleware` 뒤에 있어야 하는 것처럼, Middleware 사이에는 순서 의존성이 존재한다.

### Short-circuit

Middleware가 `get_response()`를 호출하지 않고 응답을 반환하면 안쪽 Middleware와 view는 실행되지 않는다.

```python
from django.http import JsonResponse


class MaintenanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if is_maintenance_mode():
            return JsonResponse(
                {"detail": "maintenance"},
                status=503,
            )

        return self.get_response(request)
```

점검 모드나 전역 접근 차단에는 유용하지만, 안쪽 Middleware의 로깅·정리 로직까지 건너뛸 수 있다. 조기 응답이 필요한 위치와 그 영향을 함께 확인해야 한다.

### 추가 Middleware hook

class-based Middleware는 필요에 따라 view 직전이나 예외 처리 시점에 개입할 수 있다.

- `process_view()`: view를 호출하기 직전에 실행한다.
- `process_exception()`: view에서 예외가 발생했을 때 실행한다.
- `process_template_response()`: template response가 반환된 뒤 실행한다.

응답 단계와 예외 처리 hook은 Middleware 순서의 영향을 받으므로, 특정 Middleware가 응답을 반환하거나 예외를 처리한 뒤 어떤 계층이 실행되는지 확인해야 한다.

### Middleware의 적합한 역할

- 보안 헤더와 공통 CORS 정책
- session과 인증 기반 정보 구성
- request ID, 공통 로깅, 요청 시간 측정
- 전역적인 접근 차단과 공통 캐시 정책
- 모든 endpoint에 동일하게 적용해야 하는 처리

## Middleware의 트레이드오프

- 모든 요청에 실행되므로 작은 비용도 전체 트래픽에 누적된다.
- 무거운 DB 조회나 외부 API 호출을 넣으면 정적 파일·health check까지 느려질 수 있다.
- 등록 순서에 대한 암묵적 의존성이 늘어난다.
- 특정 URL의 비즈니스 규칙을 넣으면 실제 실행 위치를 찾기 어려워진다.
- 동기 Middleware와 비동기 view를 섞으면 sync/async 변환 비용이 발생할 수 있다.
- streaming response는 일반 응답처럼 본문 전체를 메모리에 올릴 수 없으므로 별도 처리가 필요하다.

Middleware에는 횡단 관심사를 두고, 주문 상태 변경처럼 특정 도메인에만 필요한 규칙은 view나 service 계층에 두는 편이 테스트와 변경에 유리하다.

## Signal이란

Django Signal은 한 컴포넌트가 특정 사건을 알리고, 등록된 receiver가 그 사건에 반응하도록 하는 observer 방식의 기능이다.

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Post


@receiver(post_save, sender=Post)
def handle_post_saved(sender, instance, created, **kwargs):
    if created:
        record_post_created(instance.id)
```

개념적인 실행 흐름은 다음과 같다.

```text
post.save()
  -> SQL INSERT 또는 UPDATE
  -> post_save 전송
      -> receiver A 실행
      -> receiver B 실행
  -> save() 호출 흐름으로 복귀
```

Signal은 메시지 브로커나 백그라운드 작업 큐가 아니다. 기본적인 signal 전송은 현재 호출 흐름에서 receiver를 실행하므로 receiver가 느리거나 예외를 발생시키면 원래 요청에도 영향을 줄 수 있다.

## Signal의 장점

- Django나 서드파티 앱의 이벤트를 확장할 수 있다.
- 여러 앱이 하나의 이벤트를 구독할 수 있다.
- sender가 모든 receiver를 직접 import하지 않아도 된다.
- 감사 로그, 캐시 무효화, 검색 인덱스 갱신 같은 부가 동작을 분리할 수 있다.

재사용 가능한 앱이나 플러그인 구조에서 특정 이벤트에 반응해야 하지만 sender가 receiver의 구현을 알면 안 되는 경우에 특히 유용하다.

## Signal을 남용하면 생기는 문제

### 실행 흐름이 숨겨진다

`post.save()`만 보고는 저장 직후 이메일, 통계 갱신, 캐시 삭제, 외부 API 호출이 실행되는지 알기 어렵다. 코드 검색과 디버깅 범위가 넓어지고 변경 영향도 파악하기 어려워진다.

### 핵심 비즈니스 순서가 불명확해진다

결제 완료 후 주문 상태 변경, 재고 차감, 알림 발송처럼 순서와 실패 처리가 중요한 로직을 여러 receiver로 나누면 전체 transaction 흐름이 보이지 않는다. receiver 등록 순서에 의존하는 설계도 피해야 한다.

### commit 이전에 실행될 수 있다

`post_save`는 데이터베이스 transaction이 commit되었다는 뜻이 아니다. commit 전에 외부 API나 작업 큐를 호출하면 이후 rollback이 발생했을 때 외부 시스템에 잘못된 이벤트가 남을 수 있다.

commit 이후 실행되어야 하는 작업은 `transaction.on_commit()`으로 미룬다.

```python
from django.db import transaction
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Post


@receiver(post_save, sender=Post)
def enqueue_post_created(sender, instance, created, **kwargs):
    if not created:
        return

    transaction.on_commit(
        lambda: publish_post_created.delay(instance.id)
    )
```

### 모든 ORM 쓰기에서 호출되는 것은 아니다

`QuerySet.update()`는 모델 인스턴스의 `save()`를 호출하지 않으므로 `pre_save`와 `post_save`가 발생하지 않는다. `bulk_create()`도 각 객체의 `save()`와 관련 signal을 호출하지 않는다.

데이터 정합성에 필요한 로직을 Signal에만 의존하면 일괄 업데이트, 관리 명령, 데이터 마이그레이션에서 해당 로직이 누락될 수 있다.

### 중복 등록과 오류 전파

receiver 연결 코드가 여러 번 실행되면 같은 receiver가 중복 호출될 수 있다. `AppConfig.ready()`에서 연결 지점을 관리하고, 직접 연결할 때는 `dispatch_uid`를 사용할 수 있다.

```python
post_save.connect(
    handle_post_saved,
    sender=Post,
    dispatch_uid="posts.handle_post_saved",
)
```

receiver가 외부 API를 동기 호출하면 원 요청이 느려지고, 예외가 원래 저장 흐름까지 실패시킬 수 있다. 재시도와 장애 격리가 필요한 작업은 commit 이후 작업 큐로 전달하는 편이 적합하다.

## Signal을 사용할지 판단하는 기준

Signal을 사용하기 좋은 경우는 다음과 같다.

- 여러 앱이 관심을 가질 수 있는 부가 이벤트
- Django나 서드파티 앱의 동작을 확장해야 하는 경우
- 호출 순서가 핵심이 아니고 실패 영향이 제한적인 처리
- sender와 receiver를 느슨하게 연결해야 하는 플러그인 구조

반대로 순서, transaction, 실패 처리가 중요한 핵심 비즈니스 로직은 service 함수에 명시하는 편이 낫다.

```python
from django.db import transaction


@transaction.atomic
def publish_post(post):
    post.mark_as_published()
    post.save(update_fields=["status", "published_at"])
    create_publication_history(post)

    transaction.on_commit(
        lambda: notify_subscribers.delay(post.id)
    )
```

상태 변경과 이력 저장은 같은 transaction에 포함하고, 외부 알림은 commit 이후 실행된다는 흐름을 한 곳에서 확인할 수 있다.

## 정리

- Middleware는 요청 시 등록 순서, 응답 시 역순으로 실행되는 전역 처리 계층이다.
- `get_response()`를 호출하지 않는 Middleware는 안쪽 Middleware와 view를 건너뛴다.
- Middleware에는 보안, 인증 기반 정보, request ID, 공통 로깅 같은 횡단 관심사를 둔다.
- Signal은 이벤트에 반응하는 확장 지점이지만 기본적으로 현재 호출 흐름 안에서 실행된다.
- `post_save`는 transaction commit 완료를 의미하지 않으므로 외부 부작용은 `transaction.on_commit()` 이후로 미루는 것이 안전하다.
- `update()`와 `bulk_create()`에서는 일반적인 model save signal이 실행되지 않는다.
- 핵심 비즈니스 로직은 Signal보다 명시적인 service 함수로 관리하는 편이 추적과 테스트에 유리하다.

## 학습 체크리스트

- [ ] Middleware 요청·응답 실행 순서와 등록 순서를 설명할 수 있는가?
- [ ] Middleware가 short-circuit할 때 실행되지 않는 계층을 알고 있는가?
- [ ] 모든 요청에 적용할 로직과 특정 도메인 로직을 구분했는가?
- [ ] Signal receiver가 동기 호출 흐름에서 실행된다는 점을 고려했는가?
- [ ] commit 이후 실행해야 하는 외부 작업을 `transaction.on_commit()`으로 감쌌는가?
- [ ] bulk 작업에서 Signal이 실행되지 않을 수 있음을 확인했는가?
- [ ] 핵심 비즈니스 로직이 Signal에 숨겨져 있지 않은가?

## 참고

- [Django Middleware](https://docs.djangoproject.com/en/5.2/topics/http/middleware/)
- [Django Signals](https://docs.djangoproject.com/en/5.2/topics/signals/)
- [Django Transactions: `on_commit()`](https://docs.djangoproject.com/en/5.2/topics/db/transactions/#performing-actions-after-commit)
