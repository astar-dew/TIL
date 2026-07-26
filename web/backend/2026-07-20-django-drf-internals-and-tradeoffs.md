---
layout: post
title: "Django와 DRF 내부 동작: N+1 최적화, Middleware, Signal"
date: 2026-07-20
categories: [web, backend]
tags: [django, drf, orm, n-plus-one, middleware, signal]
description: "Django와 DRF에서 N+1 쿼리가 발생하는 원인과 select_related, prefetch_related의 해결 방식, Middleware와 Signal의 내부 흐름 및 트레이드오프를 정리한다."
---

## 학습 목적

Django와 Django REST Framework(DRF)는 많은 기능을 추상화해서 빠르게 API를 개발할 수 있게 한다. 하지만 ORM 쿼리가 실행되는 시점, Middleware가 요청과 응답을 감싸는 순서, Signal receiver가 호출되는 위치를 모르면 성능 문제와 추적하기 어려운 부작용이 생길 수 있다.

이 글에서는 먼저 N+1 문제가 왜 발생하는지 살펴본 뒤 `select_related()`와 `prefetch_related()`가 이를 어떻게 해결하는지 정리한다. 이후 Django의 요청 처리 흐름과 Middleware, Signal의 장단점을 살펴본다.

## 예제 모델

게시글 목록 API가 작성자와 태그를 함께 반환한다고 가정한다.

```python
from django.db import models


class Author(models.Model):
    name = models.CharField(max_length=100)


class Tag(models.Model):
    name = models.CharField(max_length=50)
    is_public = models.BooleanField(default=True)


class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,
        related_name="posts",
    )
    tags = models.ManyToManyField(Tag, related_name="posts")
```

DRF serializer는 관계 필드에 접근한다.

```python
from rest_framework import serializers

from .models import Post


class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.name")
    tags = serializers.SlugRelatedField(
        many=True,
        read_only=True,
        slug_field="name",
    )

    class Meta:
        model = Post
        fields = ["id", "title", "author_name", "tags"]
```

## N+1 문제란 무엇인가

N+1 문제는 목록을 가져오는 쿼리 1번과 목록의 각 객체가 관계를 조회하는 추가 쿼리 N번이 실행되는 현상이다.

```python
posts = Post.objects.all()

for post in posts:
    print(post.author.name)
```

게시글이 10개라면 일반적으로 다음과 같은 쿼리 흐름이 만들어진다.

```text
1. SELECT * FROM post;                  -- 게시글 목록 1번
2. SELECT * FROM author WHERE id = ?;  -- 첫 번째 게시글의 작성자
3. SELECT * FROM author WHERE id = ?;  -- 두 번째 게시글의 작성자
...
11. SELECT * FROM author WHERE id = ?; -- 열 번째 게시글의 작성자
```

결과적으로 전체 쿼리 수는 `1 + N`이 된다. 작성자뿐 아니라 태그도 게시글마다 조회하면 `1 + N + N`, 즉 `1 + 2N` 형태로 늘어날 수 있다.

### 왜 N+1이 발생하는가

Django `QuerySet`은 지연 평가된다. `Post.objects.all()`을 작성하는 순간에는 SQL이 바로 실행되지 않고, 반복하거나 `list()`로 변환하는 등 결과가 필요해지는 시점에 실행된다.

관계 객체도 필요한 시점에 가져온다.

```text
Post.objects.all()
    |
    +-- QuerySet 평가: Post 목록 조회
            |
            +-- post.author 접근: 해당 Author 조회
            |
            +-- post.tags.all() 접근: 해당 Post의 Tag 목록 조회
```

이 동작은 관계를 사용하지 않을 때 불필요한 데이터를 가져오지 않는다는 장점이 있다. 문제는 목록의 모든 객체에서 같은 관계를 사용할 때도 각 객체의 속성 접근이 별도 쿼리를 만들 수 있다는 점이다.

Django 모델 인스턴스는 한 번 조회한 단일 관계를 해당 인스턴스에 캐시한다. 하지만 서로 다른 `Post` 인스턴스가 같은 작성자를 가리킨다고 해서 모든 인스턴스가 관계 캐시를 공유하는 것은 아니다.

### DRF에서 더 쉽게 발생하는 이유

DRF serializer는 응답 데이터를 만드는 과정에서 선언된 필드를 하나씩 읽는다. 다음 필드는 게시글마다 관계 객체를 사용한다.

```python
author_name = serializers.CharField(source="author.name")
tags = serializers.SlugRelatedField(
    many=True,
    read_only=True,
    slug_field="name",
)
```

DRF는 serializer 구조를 분석해서 `select_related()`나 `prefetch_related()`를 자동 적용하지 않는다. serializer의 필드, `SerializerMethodField`, 모델 property 안에서 관계에 접근하면 화면에서는 정상 응답이 나오지만 내부에서는 쿼리가 반복될 수 있다.

따라서 관계를 어떻게 가져올지는 view의 queryset을 만드는 개발자가 결정해야 한다.

## N+1이 성능 병목이 되는 이유

DB 쿼리는 Python 속성 접근보다 훨씬 비싸다. 애플리케이션과 DB 사이에는 네트워크 왕복, SQL 파싱, 실행 계획 수립, 디스크 또는 메모리 접근, 결과 전송 과정이 존재한다.

게시글 수가 늘어나면 쿼리 수도 함께 증가한다.

| 게시글 수 | 작성자만 지연 조회 | 작성자와 태그 지연 조회 |
| ---: | ---: | ---: |
| 10개 | 11개 | 21개 |
| 100개 | 101개 | 201개 |
| 1,000개 | 1,001개 | 2,001개 |

개발 환경에서는 데이터가 적고 DB가 같은 장비에 있어 문제가 잘 드러나지 않는다. 운영 환경에서는 네트워크 지연과 동시 요청이 겹쳐 응답 시간이 길어지고 DB 연결 풀이 빠르게 소진될 수 있다.

## 해결 1: `select_related()`

`select_related()`는 SQL JOIN을 사용해서 기준 모델과 단일 관계의 데이터를 한 쿼리로 가져온다.

```python
posts = Post.objects.select_related("author")

for post in posts:
    print(post.author.name)
```

개념적으로 생성되는 SQL은 다음과 같다.

```sql
SELECT
    post.id,
    post.title,
    post.author_id,
    author.id,
    author.name
FROM post
INNER JOIN author
    ON post.author_id = author.id;
```

### 어떻게 해결되는가

기존에는 `Post` 목록을 조회한 뒤 `post.author`를 처음 읽을 때마다 작성자 쿼리가 실행되었다. `select_related("author")`를 적용하면 처음 쿼리의 JOIN 결과에 작성자 데이터가 이미 들어 있다.

```text
적용 전
Post 조회 1번 + 각 Post의 Author 조회 N번 = 1 + N

적용 후
Post와 Author를 JOIN해서 조회 1번 = 1
```

Django는 JOIN 결과로 `Post`와 `Author` 객체를 만들고 각 `Post`의 관계 캐시에 작성자를 채운다. 이후 `post.author`에 접근해도 추가 쿼리가 발생하지 않는다.

### 적합한 관계

`select_related()`는 결과 하나에 관련 객체가 최대 하나인 관계에 적합하다.

- `ForeignKey`
- `OneToOneField`
- 역방향 `OneToOneField`

### 트레이드오프

- JOIN하는 테이블과 컬럼이 많아지면 SQL이 복잡해진다.
- 넓은 테이블을 여러 개 JOIN하면 한 행의 데이터 크기가 커진다.
- 사용하지 않는 관계까지 가져오면 DB와 네트워크 비용만 늘어난다.
- 인자 없이 `select_related()`를 호출하면 불필요한 관계까지 따라갈 수 있으므로 필요한 필드를 명시하는 편이 낫다.
- 다대다나 역방향 ForeignKey처럼 한 객체에 여러 결과가 연결되는 관계에는 사용할 수 없다.

## 해결 2: `prefetch_related()`

`prefetch_related()`는 기준 객체와 관련 객체를 각각 조회한 뒤 Python에서 관계를 연결한다.

```python
posts = Post.objects.prefetch_related("tags")

for post in posts:
    print([tag.name for tag in post.tags.all()])
```

개념적으로 다음 두 쿼리가 실행된다.

```sql
SELECT * FROM post;

SELECT tag.*, post_tags.post_id
FROM tag
INNER JOIN post_tags
    ON tag.id = post_tags.tag_id
WHERE post_tags.post_id IN (1, 2, 3, ...);
```

### 어떻게 해결되는가

기존에는 게시글마다 `post.tags.all()`이 별도 쿼리를 실행했다. `prefetch_related("tags")`를 적용하면 Django가 먼저 모든 게시글을 가져오고, 해당 게시글 ID를 모아 태그를 한 번에 조회한다.

```text
적용 전
Post 조회 1번 + 각 Post의 Tag 조회 N번 = 1 + N

적용 후
Post 조회 1번 + 전체 Post의 Tag 일괄 조회 1번 = 2
```

두 번째 쿼리 결과는 Python에서 `post_id` 기준으로 묶이고 각 `Post`의 관계 캐시에 저장된다. 따라서 `post.tags.all()`은 DB를 다시 조회하지 않고 미리 가져온 결과를 사용한다.

### 적합한 관계

`prefetch_related()`는 한 객체에 여러 관련 객체가 연결될 수 있는 관계에 적합하다.

- `ManyToManyField`
- 역방향 `ForeignKey`
- `GenericRelation`
- 필터링하거나 추가 최적화가 필요한 관계

### `Prefetch`로 가져올 데이터 제어하기

모든 관련 객체가 아니라 공개된 태그만 이름순으로 가져오고 싶다면 `Prefetch` 객체를 사용할 수 있다.

```python
from django.db.models import Prefetch

from .models import Post, Tag


posts = Post.objects.prefetch_related(
    Prefetch(
        "tags",
        queryset=Tag.objects.filter(is_public=True).order_by("name"),
        to_attr="public_tags",
    )
)
```

`to_attr`를 사용하면 필터링한 결과를 `post.public_tags`에 리스트로 저장한다. 원래 관계 manager의 전체 데이터와 필터링된 결과가 섞이는 것을 피할 수 있다.

### 트레이드오프

- 기준 쿼리와 관계 쿼리가 별도로 실행되므로 두 쿼리 사이에 데이터가 변경될 수 있다.
- 기준 객체와 관련 객체를 메모리에 미리 올리므로 결과가 크면 메모리 사용량이 증가한다.
- 기준 객체가 매우 많으면 큰 `IN (...)` 절이 생성되어 DB 성능에 영향을 줄 수 있다.
- prefetch한 `post.tags.all()` 대신 `post.tags.filter(...)`를 호출하면 새로운 쿼리가 실행된다.
- 사용하지 않을 관계를 prefetch하면 쿼리와 메모리 비용만 추가된다.

## 두 방식을 함께 적용하기

게시글 목록에서 작성자와 태그를 모두 반환한다면 두 방식을 조합한다.

```python
posts = (
    Post.objects
    .select_related("author")
    .prefetch_related("tags")
)
```

쿼리 흐름은 다음과 같이 바뀐다.

```text
1. Post + Author JOIN 쿼리
2. 전체 Post에 연결된 Tag 일괄 쿼리
```

게시글이 10개든 1,000개든 기본 쿼리 수는 관계 구조가 같다면 2개로 유지된다. 다만 페이지네이션의 전체 개수 조회, 추가 serializer 필드, 필터와 property 접근에 따라 실제 쿼리 수는 달라질 수 있다.

## DRF ViewSet에서 적용하기

DRF 목록 API에서는 `get_queryset()`에 최적화를 명시하는 것이 일반적이다.

```python
from rest_framework.viewsets import ReadOnlyModelViewSet

from .models import Post
from .serializers import PostSerializer


class PostViewSet(ReadOnlyModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        return (
            Post.objects
            .select_related("author")
            .prefetch_related("tags")
        )
```

요청별 사용자, query parameter, action에 따라 관계가 달라진다면 `get_queryset()`에서 조건을 나눌 수 있다. 이때 serializer가 실제로 읽는 관계와 queryset 최적화가 함께 변경되어야 한다.

```python
def get_queryset(self):
    queryset = Post.objects.select_related("author")

    if self.action == "retrieve":
        return queryset.prefetch_related("tags", "comments__author")

    return queryset.prefetch_related("tags")
```

목록과 상세 API가 서로 다른 관계를 사용한다면 무조건 가장 큰 queryset을 공통으로 적용하기보다 action별로 필요한 데이터만 가져오는 편이 낫다.

## N+1 문제를 확인하는 방법

쿼리 최적화는 코드 모양이 아니라 실제 쿼리 수와 실행 시간으로 검증해야 한다.

### Django Debug Toolbar

개발 환경에서 요청별 SQL, 중복 쿼리, 실행 시간을 확인할 수 있다. 운영 환경에는 그대로 노출하지 않는다.

### 테스트로 쿼리 수 고정하기

`assertNumQueries()`를 사용하면 serializer 변경으로 N+1 문제가 다시 생기는 것을 테스트에서 감지할 수 있다.

```python
from django.test import TestCase

from .models import Author, Post, Tag
from .serializers import PostSerializer


class PostSerializerQueryTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        author = Author.objects.create(name="Django")
        tag = Tag.objects.create(name="ORM")

        for index in range(3):
            post = Post.objects.create(
                title=f"Post {index}",
                author=author,
            )
            post.tags.add(tag)

    def test_post_list_uses_two_queries(self):
        queryset = (
            Post.objects
            .select_related("author")
            .prefetch_related("tags")
        )

        with self.assertNumQueries(2):
            posts = list(queryset)
            PostSerializer(posts, many=True).data
```

테스트 데이터가 1개일 때만 확인하면 N+1이 잘 드러나지 않을 수 있다. 여러 게시글과 중복 작성자, 여러 태그를 포함한 데이터를 만들어 쿼리 수가 데이터 개수에 따라 증가하지 않는지 확인해야 한다.

### 실행 계획 확인하기

쿼리 수를 줄였더라도 하나의 JOIN 쿼리가 너무 무거울 수 있다. `QuerySet.explain()`이나 DB의 `EXPLAIN`을 사용해 인덱스 사용 여부, JOIN 방식, 예상 행 수를 확인한다.

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

DRF View는 Django View 위에서 다음과 같은 API 처리 단계를 추가한다.

```text
HttpRequest
  -> DRF Request 변환
  -> 인증
  -> 권한
  -> throttling
  -> action/list/retrieve 등 handler
  -> Serializer
  -> Renderer
  -> Response
```

이 흐름을 알아야 인증, 로깅, 트랜잭션, 예외 처리, 쿼리 최적화를 어느 계층에 둘지 판단할 수 있다.

## Middleware 처리 순서

Middleware는 요청과 응답의 공통 관심사를 처리한다. `settings.py`의 `MIDDLEWARE` 목록 순서가 실행 순서를 결정한다.

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "project.middleware.RequestLoggingMiddleware",
]
```

요청은 위에서 아래로 이동하고, 응답은 아래에서 위로 돌아온다.

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
Security
응답
```

`AuthenticationMiddleware`는 session을 사용해 `request.user`를 구성하므로 `SessionMiddleware` 뒤에 있어야 한다. 이런 의존성을 무시하면 필요한 속성이 없거나 인증 상태가 올바르게 구성되지 않을 수 있다.

### 기본 Middleware 형태

```python
class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # 요청 방향: 등록 순서대로 실행
        response = self.get_response(request)
        # 응답 방향: 등록 역순으로 실행
        return response
```

`self.get_response(request)` 호출 전은 요청 처리 구간이고, 호출 후는 응답 처리 구간이다.

### Short-circuit

Middleware가 `get_response()`를 호출하지 않고 바로 `HttpResponse`를 반환하면 안쪽 Middleware와 View는 실행되지 않는다.

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

이 방식은 점검 모드, IP 차단, 일부 캐시 응답에 유용하다. 하지만 예상하지 못한 위치에서 응답이 끝나면 안쪽 Middleware의 로깅이나 정리 로직도 실행되지 않을 수 있으므로 순서와 책임을 명확히 해야 한다.

### Middleware의 적합한 역할

- 보안 헤더
- 세션과 인증 기반 정보 구성
- request ID와 공통 로깅
- 공통 CORS 또는 캐시 정책
- 요청 시간 측정
- 전역적인 접근 차단

### Middleware 남용 시 문제

- 모든 요청에 실행되어 작은 비용도 전체 트래픽에 누적된다.
- URL별 비즈니스 규칙이 들어가면 실행 흐름을 찾기 어렵다.
- DB 조회를 수행하면 정적 파일이나 health check에도 불필요한 쿼리가 생길 수 있다.
- 등록 순서에 대한 암묵적 의존성이 늘어난다.
- streaming response와 async view에서는 일반 응답과 다른 동작 특성을 고려해야 한다.

Middleware에는 여러 endpoint에 공통으로 적용되는 횡단 관심사를 두고, 특정 기능의 비즈니스 로직은 view나 service 계층에 두는 편이 명확하다.

## Signal의 동작 방식

Django Signal은 sender에서 특정 사건이 발생했음을 알리고 등록된 receiver가 반응하도록 하는 observer 방식의 기능이다.

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Post


@receiver(post_save, sender=Post)
def handle_post_saved(sender, instance, created, **kwargs):
    if created:
        record_post_created(instance.id)
```

개념적인 흐름은 다음과 같다.

```text
post.save()
  -> SQL INSERT 또는 UPDATE
  -> post_save 전송
      -> receiver A 실행
      -> receiver B 실행
  -> save() 호출 흐름으로 복귀
```

Signal은 메시지 브로커나 백그라운드 작업 큐가 아니다. 일반적인 signal 전송은 현재 호출 흐름 안에서 receiver를 실행하므로 receiver가 느리거나 예외를 발생시키면 원래 요청에도 영향을 준다.

### Signal의 장점

- 프레임워크나 외부 앱의 이벤트에 별도 코드가 반응할 수 있다.
- 여러 앱이 하나의 이벤트를 구독하는 확장 지점을 만들 수 있다.
- sender가 모든 receiver를 직접 import하지 않아도 된다.
- 감사 로그나 캐시 무효화처럼 부가 동작을 분리할 수 있다.

## Signal의 단점과 남용 문제

### 실행 흐름이 숨겨진다

`post.save()`만 보고 이메일 발송, 통계 갱신, 외부 API 호출이 실행된다는 사실을 알기 어렵다. 코드 검색과 디버깅 범위가 넓어지고, 변경의 영향도 파악하기 어려워진다.

### 핵심 비즈니스 로직의 순서가 불명확해진다

결제 완료 후 주문 상태 변경, 재고 차감, 알림 발송처럼 순서와 실패 처리가 중요한 작업을 여러 signal receiver로 나누면 전체 트랜잭션 흐름이 보이지 않는다. receiver 등록 순서에 의존하는 설계도 피해야 한다.

### 트랜잭션 commit 전에 실행될 수 있다

`post_save`라는 이름은 DB transaction이 commit되었다는 뜻이 아니다. 외부 API 호출이나 작업 큐 등록이 commit 전에 실행되면 이후 transaction rollback 시 외부 시스템과 DB 상태가 달라질 수 있다.

commit 이후 실행되어야 하는 작업은 `transaction.on_commit()`을 사용한다.

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

### 모든 ORM 쓰기 방식에서 호출되지 않는다

`QuerySet.update()`는 모델의 `save()`를 호출하지 않으므로 `pre_save`와 `post_save`가 발생하지 않는다. `bulk_create()`도 개별 객체의 `save()`와 관련 signal을 호출하지 않는다.

따라서 데이터 정합성을 signal에만 의존하면 일괄 업데이트나 관리 명령에서 로직이 빠질 수 있다.

### 중복 등록과 테스트 문제가 생길 수 있다

receiver 모듈이 여러 번 import되거나 앱 초기화 과정에서 연결 코드가 반복되면 같은 receiver가 중복 호출될 수 있다. `AppConfig.ready()`에서 receiver를 import하고, 직접 `connect()`할 때는 필요하면 `dispatch_uid`를 사용한다.

```python
post_save.connect(
    handle_post_saved,
    sender=Post,
    dispatch_uid="posts.handle_post_saved",
)
```

### 오류와 지연이 원 요청으로 전파된다

receiver에서 외부 API를 동기 호출하면 응답 시간이 늘어난다. receiver가 예외를 발생시키면 signal 전송 방식에 따라 원래 저장이나 요청이 실패할 수 있다. 외부 연동처럼 재시도와 장애 격리가 필요한 작업은 transaction commit 이후 작업 큐로 전달하는 편이 낫다.

## Signal을 사용해도 좋은 경우

- 여러 앱이 관심을 가질 수 있는 부가 이벤트
- Django나 서드파티 앱의 동작을 확장해야 하는 경우
- 호출 순서가 핵심이 아니고 실패 영향이 제한적인 처리
- 명시적인 service 의존성을 만들기 어려운 플러그인 구조

## 직접 호출이 더 나은 경우

핵심 비즈니스 로직은 service 함수에 순서를 명시하는 편이 이해하기 쉽다.

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

이 코드는 상태 변경과 이력 저장이 같은 transaction에 포함되고, 알림은 commit 이후 실행된다는 흐름을 한 곳에서 확인할 수 있다.

## 선택 기준 정리

| 문제 | 우선 선택 | 내부 동작 | 주요 비용 |
| --- | --- | --- | --- |
| ForeignKey, OneToOne N+1 | `select_related()` | SQL JOIN으로 한 번에 조회 | 복잡한 JOIN, 넓은 결과 행 |
| ManyToMany, 역방향 ForeignKey N+1 | `prefetch_related()` | 별도 쿼리 후 Python에서 연결 | 메모리, 큰 `IN` 절 |
| 전체 요청 공통 처리 | Middleware | 요청 순서, 응답 역순으로 View를 감쌈 | 전체 요청 비용, 순서 의존 |
| 여러 앱의 부가 이벤트 구독 | Signal | sender가 receiver를 호출 | 숨겨진 흐름, 오류 전파 |
| 순서와 transaction이 중요한 핵심 로직 | service 직접 호출 | 호출 관계를 코드에 명시 | 계층 간 의존성을 직접 관리 |
| 오래 걸리거나 재시도가 필요한 외부 작업 | 작업 큐 | commit 후 비동기 worker 실행 | 큐 운영과 결과 추적 |

## 정리

- N+1은 목록 쿼리 후 각 객체의 관계를 지연 조회하면서 쿼리 수가 데이터 개수에 비례해 늘어나는 문제다.
- `select_related()`는 단일 관계를 SQL JOIN으로 가져와 `1 + N`을 1개 쿼리로 줄인다.
- `prefetch_related()`는 다중 관계를 별도 쿼리로 일괄 조회해 `1 + N`을 보통 2개 쿼리로 줄인다.
- DRF는 serializer 관계를 자동으로 최적화하지 않으므로 queryset에서 직접 처리해야 한다.
- Middleware는 요청 때 `MIDDLEWARE` 순서, 응답 때 역순으로 실행된다.
- Signal은 확장 지점에는 유용하지만 핵심 비즈니스 흐름에 남용하면 추적과 실패 처리가 어려워진다.
- 쿼리 최적화는 데이터 개수, 쿼리 수, 실행 계획, 메모리 사용량을 함께 측정해야 한다.

## 학습 체크리스트

- [ ] N+1 쿼리가 발생하는 ORM 속성 접근을 찾을 수 있는가?
- [ ] 관계의 cardinality에 따라 `select_related()`와 `prefetch_related()`를 선택할 수 있는가?
- [ ] DRF serializer가 읽는 관계와 view의 queryset을 함께 검토했는가?
- [ ] 데이터 개수가 늘어도 쿼리 수가 유지되는 테스트가 있는가?
- [ ] Middleware의 요청과 응답 실행 순서를 설명할 수 있는가?
- [ ] Signal receiver가 transaction commit 전에 실행될 수 있음을 고려했는가?
- [ ] 핵심 비즈니스 로직이 Signal에 숨겨져 있지 않은가?

## 참고

- [Django QuerySet API: `select_related()`와 `prefetch_related()`](https://docs.djangoproject.com/en/5.2/ref/models/querysets/)
- [Django Database access optimization](https://docs.djangoproject.com/en/5.2/topics/db/optimization/)
- [Django Middleware](https://docs.djangoproject.com/en/5.2/topics/http/middleware/)
- [Django Signals](https://docs.djangoproject.com/en/5.2/topics/signals/)
- [Django Transactions: `on_commit()`](https://docs.djangoproject.com/en/5.2/topics/db/transactions/#performing-actions-after-commit)
- [DRF Generic views: Avoiding N+1 Queries](https://www.django-rest-framework.org/api-guide/generic-views/#avoiding-n1-queries)
- [DRF Serializer relations](https://www.django-rest-framework.org/api-guide/relations/)
