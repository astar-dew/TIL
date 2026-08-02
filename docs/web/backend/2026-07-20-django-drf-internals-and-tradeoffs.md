---
title: "Django ORM과 DRF의 N+1 문제 최적화"
date: 2026-07-20
tags: [django, drf, orm, n-plus-one, database, performance]
description: "Django ORM과 DRF에서 N+1 쿼리가 발생하는 원인과 select_related, prefetch_related를 이용한 해결 방식을 정리한다."
---

## 학습 목적

Django와 Django REST Framework(DRF)는 많은 기능을 추상화해서 빠르게 API를 개발할 수 있게 한다. 하지만 ORM 쿼리가 실행되는 시점을 모르면 목록 API에서 데이터가 늘어날수록 응답이 느려지는 문제가 발생할 수 있다.

이 글에서는 N+1 문제가 왜 발생하는지 먼저 살펴본 뒤 `select_related()`와 `prefetch_related()`가 이를 어떻게 해결하는지 정리한다. DRF serializer와 queryset 최적화, 쿼리 수 검증 방법도 함께 다룬다.

## 예시 모델

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

관계는 다음과 같다.

```text
Post N --- 1 Author
Post N --- N Tag
```

작성자는 단일 객체 관계이고 태그는 여러 객체 관계이므로 각각 다른 ORM 최적화 방식을 사용한다.

## N+1 문제란

N+1 문제는 목록을 가져오는 1번의 쿼리를 실행한 뒤, 목록의 각 항목이 연관 데이터를 조회하면서 N번의 추가 쿼리를 실행하는 현상이다.

```python
posts = Post.objects.all()

for post in posts:
    print(post.title, post.author.name)
```

게시글이 100개라면 실제 쿼리는 다음처럼 늘어날 수 있다.

```text
1. Post 목록 조회                       1회
2. 첫 번째 Post의 Author 조회            1회
3. 두 번째 Post의 Author 조회            1회
...
101. 마지막 Post의 Author 조회           1회
------------------------------------------
총 쿼리 수                               1 + N회
```

객체 하나를 읽는 코드가 간단해 보여도 이 접근이 반복문이나 serializer 내부에서 반복되면 데이터 개수에 비례해 쿼리 수가 증가한다. DB가 별도 서버에 있다면 쿼리마다 네트워크 왕복 비용도 더해진다.

### Django에서 발생하는 이유

Django의 `QuerySet`은 지연 평가된다. `Post.objects.all()`을 작성하는 순간 SQL이 실행되는 것이 아니라 반복문, `list()`, `len()`, `bool()`처럼 실제 결과가 필요한 시점에 실행된다.

연관 객체도 기본적으로 필요한 시점에 조회된다.

```python
posts = Post.objects.all()  # QuerySet 구성
list(posts)                 # Post 목록 쿼리 실행
post.author                 # 관련 Author 쿼리 실행 가능
```

한 모델 인스턴스에서 이미 읽은 관계는 캐시될 수 있지만, 목록의 각 인스턴스가 관계를 자동으로 한 번에 공유하는 것은 아니다. 따라서 목록에서 같은 관계를 반복해서 사용하면 N+1이 발생할 수 있다.

## DRF에서 N+1이 더 쉽게 발생하는 이유

DRF serializer는 응답을 만들 때 선언된 필드와 관계를 읽는다.

```python
from rest_framework import serializers


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

`author.name`, `tags`, `SerializerMethodField`, 모델 property에서 관계를 읽으면 각 객체마다 추가 쿼리가 실행될 수 있다. DRF는 serializer 구조를 분석해 `select_related()`나 `prefetch_related()`를 자동 적용하지 않으므로, view의 queryset을 개발자가 최적화해야 한다.

## 해결 과정 1: 쿼리를 측정한다

최적화 전에 실제 SQL과 쿼리 수를 확인해야 한다. 개발 환경에서는 Django Debug Toolbar나 `connection.queries`를 사용할 수 있다.

```python
from django.db import connection

posts = list(Post.objects.all())
for post in posts:
    _ = post.author.name

print(len(connection.queries))
```

테스트에서는 `assertNumQueries()`로 N+1 재발을 확인할 수 있다.

```python
from django.test import TestCase


class PostQueryTest(TestCase):
    def test_post_list_query_count(self):
        with self.assertNumQueries(2):
            response = self.client.get("/api/posts/")

        self.assertEqual(response.status_code, 200)
```

실제 endpoint에서는 pagination count, 인증, 권한, 필터에 따른 쿼리가 추가될 수 있으므로 테스트 조건과 기대 쿼리 수를 함께 관리해야 한다. 쿼리 수를 줄인 뒤에도 `QuerySet.explain()`으로 JOIN과 인덱스 실행 계획을 확인해야 한다.

## 해결책 1: `select_related()`

`select_related()`는 `ForeignKey`나 `OneToOneField`처럼 결과가 하나로 정해지는 관계에 사용한다. 관련 테이블을 SQL `JOIN`으로 연결해 한 번의 조회 결과에 연관 객체를 포함한다.

```python
posts = Post.objects.select_related("author")

for post in posts:
    print(post.title, post.author.name)
```

개념적으로는 다음과 같은 SQL에 가깝다.

```sql
SELECT post.*, author.*
FROM post
JOIN author ON post.author_id = author.id;
```

기존에는 `post.author`를 읽을 때마다 추가 조회가 발생했지만, 이제 작성자 데이터가 처음부터 로드되어 관계 접근이 캐시된 객체를 사용한다.

```text
적용 전: Post 목록 1회 + Author별 N회 = 1 + N회
적용 후: Post와 Author를 JOIN으로 조회   = 1회
```

### `select_related()`의 트레이드오프

- JOIN 대상이 많아질수록 SQL이 복잡해지고 결과 행이 커질 수 있다.
- 다중 관계를 JOIN으로 연결하면 결과 행이 불필요하게 늘어날 수 있다.
- 사용하지 않는 관계까지 가져오지 말고 필요한 필드를 명시한다.
- `ManyToManyField`나 역방향 `ForeignKey`처럼 여러 결과가 연결되는 관계에는 적합하지 않다.

## 해결책 2: `prefetch_related()`

`prefetch_related()`는 `ManyToManyField`나 역방향 `ForeignKey`처럼 여러 객체가 연결되는 관계에 사용한다. 메인 쿼리와 관련 객체 쿼리를 별도로 실행한 뒤 Django가 Python에서 결과를 연결한다.

```python
posts = Post.objects.prefetch_related("tags")

for post in posts:
    print([tag.name for tag in post.tags.all()])
```

실행 흐름은 다음과 같다.

```text
1. Post 목록 조회                    1회
2. 모든 Post의 Tag를 일괄 조회        1회
3. Python에서 Post와 Tag 연결
--------------------------------------
총 쿼리 수                            2회
```

관련 조회는 보통 `IN (...)` 조건을 사용한다. 게시글 수가 늘어도 관계 조회가 게시글마다 반복되지 않는 것이 핵심이다.

### `prefetch_related()`의 트레이드오프

- 관련 객체를 메모리에 미리 올리므로 결과가 크면 메모리 사용량이 증가한다.
- 부모 객체가 매우 많으면 큰 `IN` 절이 생성될 수 있다.
- `post.tags.all()`처럼 prefetch된 동일한 관계를 사용해야 캐시가 재사용된다.
- `post.tags.filter(...)`처럼 다른 조건을 추가하면 새로운 쿼리가 발생할 수 있다.
- pagination으로 한 번에 조회하는 부모 객체 수를 제한하는 것이 중요하다.

## `Prefetch`로 필요한 데이터만 가져오기

필터링된 관계나 추가 최적화가 필요하면 `Prefetch`를 사용한다.

```python
from django.db.models import Prefetch


posts = Post.objects.prefetch_related(
    Prefetch(
        "tags",
        queryset=Tag.objects.filter(is_public=True).order_by("name"),
        to_attr="public_tags",
    )
)
```

`to_attr`를 사용하면 결과가 `post.public_tags` 리스트에 저장된다. 전체 `post.tags` 관계와 필터링된 결과를 구분할 수 있어 캐시 의미가 명확해진다.

## DRF View에서 적용하기

목록 API에서는 `get_queryset()`에 serializer가 사용하는 관계를 명시한다.

```python
from rest_framework.viewsets import ReadOnlyModelViewSet


class PostViewSet(ReadOnlyModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        return (
            Post.objects
            .select_related("author")
            .prefetch_related("tags")
            .order_by("-id")
        )
```

상세 API와 목록 API가 사용하는 관계가 다르다면 action별로 queryset을 나눠 필요 이상의 데이터를 가져오지 않는다. 이때 실제 serializer에 존재하는 관계만 최적화 대상에 넣어야 한다.

## 선택 기준

| 관계 또는 문제 | 우선 선택 | 내부 동작 | 주요 비용 |
| --- | --- | --- | --- |
| `ForeignKey`, `OneToOneField` | `select_related()` | SQL JOIN으로 조회 | 복잡한 JOIN, 넓은 결과 행 |
| `ManyToManyField`, 역방향 `ForeignKey` | `prefetch_related()` | 별도 쿼리 후 Python에서 연결 | 메모리, 큰 `IN` 절 |
| serializer가 읽는 관계 | view의 queryset 최적화 | 응답 생성 전 관계를 미리 로드 | serializer와 queryset 불일치 |
| 대량 목록 | pagination과 측정 | 조회 범위와 메모리 제한 | 페이지 크기와 count 쿼리 |

## 정리

- N+1은 목록 쿼리 후 각 객체의 관계를 지연 조회하면서 쿼리 수가 데이터 개수에 비례해 늘어나는 문제다.
- `select_related()`는 단일 관계를 SQL JOIN으로 가져와 `1 + N`을 1개 쿼리로 줄인다.
- `prefetch_related()`는 다중 관계를 별도 쿼리로 일괄 조회해 보통 2개 쿼리로 줄인다.
- DRF는 serializer 관계를 자동 최적화하지 않으므로 view의 queryset에서 직접 처리해야 한다.
- 최적화 전후의 SQL, 쿼리 수, 실행 계획, 메모리 사용량을 실제 데이터 조건에서 측정해야 한다.

## 학습 체크리스트

- [ ] serializer가 접근하는 ORM 관계를 찾을 수 있는가?
- [ ] 관계의 cardinality에 따라 `select_related()`와 `prefetch_related()`를 선택할 수 있는가?
- [ ] 데이터 개수가 늘어도 쿼리 수가 유지되는 테스트가 있는가?
- [ ] prefetch된 관계와 serializer가 실제로 읽는 속성이 일치하는가?
- [ ] pagination과 큰 `IN` 목록에 따른 메모리·DB 비용을 검토했는가?
- [ ] JOIN 쿼리의 실행 계획과 인덱스를 확인했는가?

## 참고

- [Django QuerySet API: `select_related()`와 `prefetch_related()`](https://docs.djangoproject.com/en/5.2/ref/models/querysets/)
- [Django Database access optimization](https://docs.djangoproject.com/en/5.2/topics/db/optimization/)
- [DRF Generic views: Avoiding N+1 Queries](https://www.django-rest-framework.org/api-guide/generic-views/#avoiding-n1-queries)
- [DRF Serializer relations](https://www.django-rest-framework.org/api-guide/relations/)
