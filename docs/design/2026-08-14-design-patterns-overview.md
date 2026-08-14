---
title: "디자인 패턴 시작하기: 분류 체계와 실무에서 자주 만나는 패턴"
date: 2026-08-14
tags: [design-pattern, oop, architecture, refactoring, solid, python, django]
description: "디자인 패턴의 세 가지 분류를 정리하고, 실무에서 자주 쓰는 패턴을 해결하려는 문제·효과·비용·쓰지 말아야 할 때 기준으로 정리한다."
---

## 학습 목적

디자인 패턴을 공부할 때 흔히 하는 실수가 **23개 패턴의 이름과 클래스 다이어그램을 외우는 것**이다. 그렇게 외운 지식은 실제 코드를 짤 때 거의 떠오르지 않고, 떠오르더라도 필요 없는 곳에 억지로 적용하게 된다.

패턴의 실질적인 가치는 두 가지다.

1. **이미 쓰고 있던 것에 이름이 생긴다.** Django의 미들웨어나 React의 훅에 이미 패턴이 들어 있다. 이름을 알면 구조를 더 빨리 파악한다.
2. **대화의 단위가 된다.** "여기 조건 분기를 인터페이스로 빼고 구현체를 갈아끼우게 하자"를 "Strategy로 빼자" 한마디로 줄일 수 있다.

이 글에서는 분류 체계를 잡은 뒤, 실무에서 실제로 자주 만나는 패턴을 **어떤 문제를 푸는가 / 무엇이 좋아지는가 / 비용은 무엇인가 / 언제 쓰지 말아야 하는가** 기준으로 정리한다.

## 패턴이 아닌 것

먼저 오해를 걷어내는 편이 낫다.

| 패턴은 | 패턴은 아니다 |
| --- | --- |
| 반복해서 나타난 문제의 해결 구조 | 설치해서 쓰는 라이브러리 |
| 언어에 독립적인 설계 어휘 | 항상 지켜야 하는 정답 |
| 트레이드오프가 있는 선택 | 코드 품질의 척도 |

패턴을 많이 쓴 코드가 좋은 코드는 아니다. **패턴은 유연성을 얻는 대신 간접 층을 하나 추가하는 거래**다. 변경될 일이 없는 곳에 적용하면 추적만 어려워진다.

## 세 가지 분류

GoF는 패턴을 **무엇을 다루는가**에 따라 셋으로 나눴다. 이 분류만 잡아도 "지금 내 문제가 어느 쪽인가"를 좁힐 수 있다.

| 분류 | 다루는 것 | 대표 질문 | 대표 패턴 |
| --- | --- | --- | --- |
| **생성 (Creational)** | 객체를 **어떻게 만들 것인가** | "이 객체를 누가, 어떤 조건으로 만들지?" | Factory Method, Builder, Singleton |
| **구조 (Structural)** | 객체를 **어떻게 조합할 것인가** | "이 둘을 어떻게 이어 붙이지?" | Adapter, Decorator, Facade, Proxy |
| **행위 (Behavioral)** | 객체가 **어떻게 협력할 것인가** | "이 책임을 누가 갖고 어떻게 전달하지?" | Strategy, Observer, Template Method, Command |

문제를 만났을 때 이 순서로 좁히면 된다.

```text
객체를 만드는 과정이 복잡하거나 조건에 따라 다르다   → 생성 패턴
서로 다른 것을 이어 붙이거나 기능을 덧씌워야 한다     → 구조 패턴
동작이 상황에 따라 달라지거나 통지가 필요하다         → 행위 패턴
```

## 실무에서 자주 만나는 패턴

전부 외울 필요는 없다. 아래 여섯 개만 알아도 대부분의 상황에서 대화가 통한다.

### Strategy — 조건 분기가 계속 늘어날 때

**해결하려는 문제**

결제 수단이 늘어날 때마다 같은 함수의 `if`가 길어진다.

```python
def process_payment(method: str, amount: int):
    if method == "card":
        ...
    elif method == "kakaopay":
        ...
    elif method == "naverpay":     # 추가할 때마다 이 함수를 수정
        ...
```

수단을 하나 추가할 때마다 **이미 동작하던 함수를 건드려야 한다**는 것이 문제다.

**적용 후**

```python
from typing import Protocol


class PaymentStrategy(Protocol):
    def pay(self, amount: int) -> str: ...


class CardPayment:
    def pay(self, amount: int) -> str:
        return f"카드 결제 {amount}원"


class KakaoPayPayment:
    def pay(self, amount: int) -> str:
        return f"카카오페이 결제 {amount}원"


STRATEGIES: dict[str, PaymentStrategy] = {
    "card": CardPayment(),
    "kakaopay": KakaoPayPayment(),
}


def process_payment(method: str, amount: int) -> str:
    return STRATEGIES[method].pay(amount)
```

| 좋아지는 것 | 비용 |
| --- | --- |
| 새 수단은 클래스 추가 + 등록만 하면 된다 | 클래스 수가 늘어난다 |
| 각 수단을 독립적으로 테스트할 수 있다 | 흐름이 한눈에 안 보인다 |
| 기존 코드를 수정하지 않는다 | 간접 참조가 늘어 추적이 번거롭다 |

**쓰지 말아야 할 때**: 분기가 두 개이고 늘어날 계획이 없을 때. `if/else` 한 줄이 더 읽기 쉽다.

### Factory — 생성 과정을 감춰야 할 때

**해결하려는 문제**: 객체를 만드는 데 조건 판단이나 준비 작업이 필요한데, 그 지식이 호출하는 쪽마다 흩어진다.

```python
def create_storage(env: str) -> Storage:
    if env == "prod":
        return S3Storage(bucket=settings.S3_BUCKET, region="ap-northeast-2")
    return LocalStorage(path="/tmp/uploads")
```

호출하는 쪽은 `create_storage("prod")`만 알면 되고, 어떤 구현체가 어떤 인자로 만들어지는지는 몰라도 된다.

| 좋아지는 것 | 비용 |
| --- | --- |
| 생성 로직이 한곳에 모인다 | 클래스/함수가 하나 더 생긴다 |
| 구현체를 바꿔도 호출부가 안 바뀐다 | 단순 생성에 쓰면 군더더기다 |
| 테스트에서 가짜 구현으로 갈아끼우기 쉽다 | |

**쓰지 말아야 할 때**: `User(name="a")`처럼 생성자 호출로 끝나는 경우.

### Singleton — 알되, 조심해서 쓸 것

인스턴스를 하나만 유지하는 패턴이다. 설정 객체, 커넥션 풀, 로거처럼 **정말 하나여야 하는 것**에 쓴다.

문제는 이것이 결국 **전역 상태**라는 점이다.

| 문제 | 설명 |
| --- | --- |
| 테스트 격리가 깨진다 | 앞 테스트가 바꾼 상태가 뒤 테스트에 남는다 |
| 의존 관계가 감춰진다 | 함수 시그니처만 봐서는 무엇에 의존하는지 모른다 |
| 동시성 문제 | 여러 스레드가 같은 인스턴스를 공유한다 |

**대안을 먼저 검토한다.** 대부분은 객체를 한 번 만들어 필요한 곳에 **주입**하면 해결된다. 프레임워크의 DI 컨테이너나 앱 설정 객체가 이미 그 역할을 한다.

### Observer — 상태 변화를 여러 곳에 알려야 할 때

**해결하려는 문제**: 주문이 완료되면 이메일 발송, 재고 차감, 통계 갱신이 필요한데, 이걸 전부 주문 처리 함수에 넣으면 결제 로직이 부가 기능에 묶인다.

주체(Subject)가 상태 변화를 알리면 등록된 관찰자(Observer)들이 각자 반응한다.

Django의 **Signal**이 이 패턴이다.

```python
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=Order)
def send_order_email(sender, instance, created, **kwargs):
    if created:
        ...
```

| 좋아지는 것 | 비용 |
| --- | --- |
| 주체가 관찰자를 몰라도 된다 | **실행 흐름이 코드에 드러나지 않는다** |
| 기능 추가 시 주체를 수정하지 않는다 | 디버깅이 어렵다 |
| 관심사를 분리할 수 있다 | 실행 순서를 보장하기 어렵다 |
| | 트랜잭션 경계와 엮이면 까다롭다 |

Django Signal의 실행 순서와 트랜잭션 문제는 [Django Middleware와 Signal의 동작 방식 및 트레이드오프](../web/backend/2026-07-10-django-middleware-signal.md)에 정리했다.

**쓰지 말아야 할 때**: 반응이 하나뿐이고 반드시 실행되어야 할 때. 그냥 함수를 직접 호출하는 편이 추적하기 쉽다.

### Adapter — 외부 것을 내 규격에 맞출 때

**해결하려는 문제**: 결제 PG사를 카카오페이에서 토스로 바꾸려는데, 두 SDK의 메서드 이름과 응답 형식이 완전히 다르다.

내 코드가 원하는 인터페이스를 정의하고, 외부 SDK를 그 인터페이스로 감싼다.

```python
class PaymentGateway(Protocol):
    def charge(self, order_id: str, amount: int) -> PaymentResult: ...


class KakaoPayAdapter:
    def __init__(self, client): self._client = client

    def charge(self, order_id: str, amount: int) -> PaymentResult:
        res = self._client.ready(partner_order_id=order_id, total_amount=amount)
        return PaymentResult(success=res["tid"] is not None, tx_id=res["tid"])
```

| 좋아지는 것 | 비용 |
| --- | --- |
| 외부 변경이 한 파일에만 닿는다 | 감싸는 코드가 추가된다 |
| 테스트에서 가짜 게이트웨이를 쓸 수 있다 | 외부 SDK의 고유 기능이 가려질 수 있다 |
| 여러 업체를 동시에 지원하기 쉽다 | |

**외부 API를 다루는 코드에는 거의 항상 쓸 만하다.** 실무에서 투자 대비 효과가 가장 확실한 패턴 중 하나다.

### Template Method — 뼈대는 같고 일부만 다를 때

전체 흐름은 상위에서 정하고, 달라지는 단계만 하위에서 채운다.

```python
class Report:
    def generate(self):          # 흐름은 고정
        data = self.fetch()
        rows = self.transform(data)
        return self.render(rows)

    def fetch(self): raise NotImplementedError
    def transform(self, data): return data
    def render(self, rows): raise NotImplementedError
```

DRF의 `ListAPIView`가 정확히 이 구조다. 요청 처리 흐름은 프레임워크가 정하고, 우리는 `get_queryset()`이나 `get_serializer_class()`만 재정의한다.

**쓰지 말아야 할 때**: 흐름 자체가 자주 바뀔 때. 상속은 결합이 강해서, 이 경우 Strategy로 조합하는 편이 낫다.

## 이미 쓰고 있는 패턴들

새로 배우기 전에, 익숙한 프레임워크에서 패턴을 찾아보면 이해가 빨라진다.

| 익숙한 기능 | 패턴 |
| --- | --- |
| Django Signal | Observer |
| Django/Express 미들웨어 | Chain of Responsibility |
| DRF Generic View | Template Method |
| Django ORM `QuerySet` 체이닝 | Builder (유사) |
| React `useState` 등 훅 | Strategy / Facade 성격 |
| React Context | Observer |
| 외부 SDK 래퍼 클래스 | Adapter |
| Python 데코레이터 | Decorator (개념적으로) |

Python의 `@decorator` 문법과 GoF의 Decorator 패턴은 이름이 같지만 **정확히 같지는 않다.** GoF Decorator는 같은 인터페이스를 유지한 채 객체를 감싸 기능을 덧씌우는 구조이고, Python 데코레이터는 함수를 감싸는 문법이다. 목적이 겹칠 뿐 범위가 다르다.

## 언제 패턴을 도입할까

패턴을 미리 깔아두는 것은 대부분 과설계로 끝난다. 다음 신호가 왔을 때 도입하는 편이 좋다.

| 신호 | 검토할 패턴 |
| --- | --- |
| 같은 조건 분기를 여러 곳에서 반복한다 | Strategy |
| 기능을 추가할 때마다 기존 함수를 수정한다 | Strategy, Factory |
| 외부 라이브러리가 코드 전반에 퍼져 있다 | Adapter |
| 한 함수가 본래 책임과 부가 작업을 함께 한다 | Observer |
| 비슷한 클래스들이 흐름만 같고 일부만 다르다 | Template Method |
| 객체 생성 코드가 여기저기 중복된다 | Factory |

반대로 **"나중에 확장될지도 모르니까"** 는 도입 근거로 약하다. 세 번째 비슷한 코드를 쓸 때 추상화해도 늦지 않다.

## SOLID와의 관계

패턴은 대체로 SOLID 원칙을 지키기 위한 **구체적인 수단**이다.

| 원칙 | 관련 패턴 |
| --- | --- |
| **O**pen-Closed (확장에 열리고 수정에 닫힘) | Strategy, Factory, Decorator |
| **D**ependency Inversion (구현이 아닌 추상에 의존) | Adapter, Factory |
| **S**ingle Responsibility (하나의 책임) | Observer, Command |
| **L**iskov Substitution (하위 타입 대체 가능) | Template Method |
| **I**nterface Segregation (인터페이스 분리) | Adapter, Facade |

원칙이 "무엇을 지향할 것인가"라면 패턴은 "그래서 어떻게 짤 것인가"에 해당한다.

## 자주 하는 오해

### "패턴을 많이 쓸수록 좋은 설계다"

패턴은 간접 층을 추가하는 비용을 치른다. 필요 없는 곳에 쓰면 읽기만 어려워진다.

### "패턴은 클래스로만 구현한다"

Python에서는 함수와 딕셔너리로 Strategy를 구현하는 편이 더 간결한 경우가 많다. 패턴은 구조이지 특정 문법이 아니다.

### "Singleton은 편하니까 자주 쓰자"

전역 상태라서 테스트 격리와 동시성 문제를 부른다. 주입으로 해결되는지 먼저 본다.

### "GoF 23개를 다 알아야 한다"

실무에서 자주 마주치는 것은 그중 일부다. 나머지는 필요할 때 찾아보면 된다.

### "Python 데코레이터 = Decorator 패턴"

이름이 같을 뿐 범위가 다르다.

## 정리

- 패턴의 가치는 암기가 아니라 **이미 쓰는 구조에 이름을 붙이고 대화 단위를 만드는 것**이다.
- 생성·구조·행위 세 분류로 문제 영역을 먼저 좁힌다.
- Strategy는 늘어나는 조건 분기를, Factory는 흩어진 생성 로직을 정리한다.
- Observer는 관심사를 분리해 주지만 실행 흐름이 코드에 드러나지 않는 비용을 치른다.
- Adapter는 외부 의존을 한곳에 가두는, 투자 대비 효과가 확실한 패턴이다.
- Template Method는 흐름이 고정되고 일부만 달라질 때 쓰며, 흐름이 자주 바뀌면 Strategy가 낫다.
- Singleton은 전역 상태이므로 주입으로 대체 가능한지 먼저 검토한다.
- 패턴은 미리 깔지 않고, 반복이 실제로 나타났을 때 도입한다.

## 학습 체크리스트

- [ ] 생성·구조·행위 분류로 문제 영역을 좁힐 수 있는가?
- [ ] Strategy가 해결하는 문제를 조건 분기 예시로 설명할 수 있는가?
- [ ] Observer의 장점과 "흐름이 안 보인다"는 비용을 함께 설명할 수 있는가?
- [ ] Singleton을 쓰기 전에 주입으로 대체 가능한지 검토했는가?
- [ ] 담당 프로젝트에서 Adapter로 감쌀 만한 외부 의존이 있는가?
- [ ] 지금 쓰는 프레임워크에서 패턴이 적용된 지점을 하나 이상 찾을 수 있는가?
- [ ] 패턴을 도입할 때 얻는 것과 치르는 비용을 함께 말할 수 있는가?

## 참고

- [Refactoring Guru — 디자인 패턴](https://refactoring.guru/ko/design-patterns)
- [Refactoring Guru — 코드 스멜](https://refactoring.guru/ko/refactoring/smells)
- [Python — `typing.Protocol`](https://docs.python.org/3/library/typing.html#typing.Protocol)
- [Django — Signals](https://docs.djangoproject.com/en/stable/topics/signals/)
- [Django REST Framework — Generic views](https://www.django-rest-framework.org/api-guide/generic-views/)
- [Martin Fowler — Refactoring](https://martinfowler.com/books/refactoring.html)
