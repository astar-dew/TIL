---
title: "Python GIL과 동시성: CPU-bound와 I/O-bound 작업의 선택 기준"
date: 2026-07-17
tags: [python, gil, concurrency, multiprocessing, asyncio, threading]
description: "CPython의 GIL이 멀티스레딩에 미치는 영향을 이해하고 CPU-bound와 I/O-bound 작업에 맞는 동시성 처리 방식을 정리한다."
---

## 학습 목적

Python의 동시성 처리는 `threading`, `multiprocessing`, `asyncio` 중 하나를 고르는 사용법만으로 끝나지 않는다. 어떤 작업이 CPU를 사용하는지, 외부 자원을 기다리는지, 그리고 CPython 인터프리터가 스레드를 어떻게 실행하는지 이해해야 올바른 방식을 선택할 수 있다.

이 글에서는 CPython의 GIL이 Python 멀티스레딩에 미치는 영향과 CPU-bound, I/O-bound 작업을 구분하는 기준을 정리한다.

## 먼저 구분할 개념

### 동시성(Concurrency)

여러 작업을 번갈아 진행해서 전체 작업이 겹쳐 진행되는 것처럼 보이게 만드는 구조다. 한 시점에 실제로 여러 CPU 코어에서 실행되는지는 별개의 문제다.

### 병렬성(Parallelism)

여러 작업을 실제로 같은 시점에 실행하는 것이다. 여러 CPU 코어를 사용하면 CPU-bound 작업을 줄이는 데 도움이 된다.

### CPU-bound와 I/O-bound

| 구분 | 작업의 병목 | 대표 사례 | 주된 선택 |
| --- | --- | --- | --- |
| CPU-bound | CPU 계산 시간 | 이미지 변환, 압축, 암호화, 대량 파싱, 순수 Python 반복문 | `multiprocessing`, `ProcessPoolExecutor` |
| I/O-bound | 네트워크나 디스크 응답 대기 | HTTP 요청, DB 조회, 파일 읽기, 외부 API 호출 | `asyncio`, `threading` |

작업이 CPU-bound인지 I/O-bound인지 모호하다면 먼저 단일 실행 시간을 측정하고, 프로파일러로 CPU 사용률과 대기 시간을 확인해야 한다. 이름이나 사용하는 라이브러리만 보고 판단하면 안 된다.

## GIL이란 무엇인가

GIL(Global Interpreter Lock)은 CPython에서 Python 객체와 바이트코드를 다루기 전에 스레드가 획득해야 하는 전역 잠금이다. 기본 GIL 빌드에서는 한 번에 하나의 스레드만 Python 바이트코드를 실행할 수 있다.

프로세스 안에 여러 스레드가 있어도 Python 코드를 실행하는 순간에는 다음과 같은 흐름이 만들어진다.

```text
스레드 A: GIL 획득 -> Python 바이트코드 실행 -> GIL 양보
스레드 B:                       GIL 획득 -> Python 바이트코드 실행
스레드 C:                                             GIL 획득 -> 실행
```

인터프리터는 실행 중인 스레드를 주기적으로 바꾸며 동시성을 제공한다. 따라서 스레드가 여러 개라는 사실만으로 Python 코드가 여러 코어에서 동시에 실행되는 것은 아니다.

### 왜 전역 잠금이 필요한가

CPython의 객체는 참조 횟수와 내부 상태를 가진다. 여러 스레드가 같은 객체를 동시에 변경하면 참조 횟수나 컨테이너의 상태가 깨질 수 있다. GIL은 이런 인터프리터 내부 상태를 보호하는 한 가지 방법으로 사용되어 왔다.

다만 GIL이 애플리케이션의 모든 연산을 안전하게 만들어 주는 것은 아니다. 예를 들어 다음 코드는 읽기와 쓰기가 하나의 논리적 작업처럼 보이지만 실제로는 여러 단계다.

```python
count = count + 1
```

두 스레드가 동시에 이 코드를 실행하면 둘 다 같은 값을 읽고 한 번씩만 증가시킬 수 있다. 공유 상태를 정확하게 변경해야 한다면 `threading.Lock`, 큐, 불변 데이터 구조 같은 별도의 동기화가 필요하다.

## GIL과 멀티스레딩의 성능 한계

다음처럼 순수 Python 코드가 계속 계산하는 작업은 CPU-bound다.

```python
def calculate(limit: int) -> int:
    total = 0
    for number in range(limit):
        total += number * number
    return total
```

이 작업을 여러 `threading.Thread`로 나누더라도 기본 CPython에서는 한 번에 하나의 스레드만 Python 바이트코드를 실행한다. 여기에 스레드 전환 비용과 공유 데이터 조정 비용까지 더해지면 단일 스레드보다 느려질 수도 있다.

반대로 스레드가 네트워크 응답이나 파일 입출력을 기다리는 동안에는 실행할 Python 코드가 없으므로 다른 스레드가 진행할 수 있다. CPython은 블로킹 I/O 주변에서 GIL을 해제하며, 일부 압축·해싱·수치 연산 라이브러리도 무거운 작업 중 GIL을 해제할 수 있다.

즉, `threading`의 핵심 장점은 순수 Python 계산을 병렬화하는 것이 아니라 기다리는 동안 다른 작업을 진행하는 데 있다.

## CPU-bound 작업: 프로세스로 분리하기

`multiprocessing`은 스레드가 아니라 별도의 프로세스를 만든다. 프로세스마다 Python 인터프리터와 메모리 공간이 분리되므로 각 프로세스가 자신의 GIL을 가지고, 여러 CPU 코어를 활용할 수 있다.

표준 라이브러리의 `ProcessPoolExecutor`를 사용하면 작업을 프로세스 풀에 전달할 수 있다.

```python
from concurrent.futures import ProcessPoolExecutor


def calculate(limit: int) -> int:
    total = 0
    for number in range(limit):
        total += number * number
    return total


if __name__ == "__main__":
    limits = [10_000_000, 10_000_000, 10_000_000, 10_000_000]

    with ProcessPoolExecutor() as executor:
        results = list(executor.map(calculate, limits))

    print(results)
```

`if __name__ == "__main__":` 가드는 프로세스가 현재 모듈을 다시 import하는 환경에서 자식 프로세스가 계속 생성되는 문제를 막기 위해 필요하다.

### 프로세스 방식의 비용

프로세스는 GIL을 우회하지만 공짜가 아니다.

- 프로세스 생성과 종료 비용이 있다.
- 프로세스끼리는 기본적으로 메모리를 공유하지 않는다.
- 작업 함수와 인자는 직렬화되어 전달되므로 큰 데이터를 복사하는 비용이 생길 수 있다.
- 작업 단위가 너무 작으면 프로세스 통신 비용이 계산 시간보다 커질 수 있다.
- 전역 변수나 열린 파일, DB 커넥션을 프로세스 간 공유한다고 가정하면 안 된다.

따라서 CPU 계산량이 충분히 크고 작업을 독립적인 단위로 나눌 수 있을 때 프로세스 풀이 효과적이다.

## I/O-bound 작업: `asyncio`와 `threading`

### `asyncio`: 협력적 동시성

`asyncio`는 이벤트 루프가 코루틴을 실행하다가 `await`를 만나는 시점에 다른 작업으로 제어를 넘기는 방식이다.

```python
import asyncio


async def fetch(resource: str) -> str:
    # 실제 코드에서는 async HTTP 클라이언트나 DB 드라이버를 사용한다.
    await asyncio.sleep(1)
    return f"completed: {resource}"


async def main() -> None:
    results = await asyncio.gather(
        fetch("user"),
        fetch("orders"),
        fetch("notifications"),
    )
    print(results)


asyncio.run(main())
```

세 작업이 각각 1초를 기다리는 상황이라면 순차 실행보다 짧은 시간에 완료될 수 있다. 중요한 점은 `await`가 실제로 비동기 대기를 등록하고 이벤트 루프에 제어권을 돌려줘야 한다는 것이다.

다음처럼 코루틴 안에서 블로킹 함수를 직접 호출하면 이벤트 루프 전체가 멈춘다.

```python
import time


async def bad_example() -> None:
    time.sleep(3)  # 이벤트 루프를 3초 동안 막는다.
```

이런 경우에는 비동기 API를 제공하는 라이브러리를 사용하거나 `asyncio.to_thread()`로 블로킹 함수를 스레드에서 실행해야 한다.

### `threading`: 블로킹 라이브러리를 함께 사용할 때

기존 HTTP 클라이언트, SDK, DB 드라이버처럼 동기 방식의 블로킹 API를 사용해야 한다면 스레드 풀을 선택할 수 있다.

```python
from concurrent.futures import ThreadPoolExecutor


def request(resource: str) -> str:
    # 실제 코드에서는 동기 HTTP 클라이언트 호출을 수행한다.
    return f"requested: {resource}"


resources = ["user", "orders", "notifications"]

with ThreadPoolExecutor(max_workers=3) as executor:
    results = list(executor.map(request, resources))

print(results)
```

스레드는 같은 프로세스의 메모리를 공유하므로 프로세스보다 데이터 전달이 간단하다. 대신 공유 가변 상태, 경쟁 상태, 데드락을 관리해야 한다. 작업 수가 무제한으로 늘어나지 않도록 스레드 풀 크기와 외부 서비스의 연결 제한도 함께 설정해야 한다.

## 어떤 방식을 선택할까

```text
작업의 병목은 무엇인가?
├─ CPU 계산
│  ├─ 순수 Python 코드가 대부분인가? -> multiprocessing / ProcessPoolExecutor
│  ├─ GIL을 해제하는 네이티브 라이브러리인가? -> 라이브러리의 병렬 처리 방식 확인
│  └─ 데이터 전달 비용이 큰가? -> 프로세스 분할보다 알고리즘·배치 크기·전용 워커 검토
└─ 외부 응답 대기
   ├─ 비동기 API를 제공하는가? -> asyncio
   ├─ 동기 블로킹 API만 제공하는가? -> threading / ThreadPoolExecutor
   └─ 작업이 크고 독립적인가? -> 큐 기반 백그라운드 워커 검토
```

| 상황 | 적합한 방식 | 이유 | 주의점 |
| --- | --- | --- | --- |
| 순수 Python으로 큰 계산을 여러 조각으로 처리 | `ProcessPoolExecutor` | 프로세스별 GIL로 여러 코어 사용 | 직렬화와 프로세스 비용 |
| 비동기 HTTP 요청을 많이 처리 | `asyncio` | 적은 스레드로 많은 대기 작업 관리 | 모든 호출이 비동기여야 함 |
| 동기 HTTP/DB 라이브러리를 병렬 호출 | `ThreadPoolExecutor` | 블로킹 대기 중 다른 스레드 진행 | 공유 상태와 연결 수 제한 |
| 단일 작업이 매우 크고 장시간 실행 | 별도 워커 또는 작업 큐 | 요청 처리 프로세스와 계산 분리 | 운영 복잡도와 결과 추적 |

## 자주 하는 오해

### “스레드가 여러 개면 CPU 계산도 빨라진다”

기본 CPython에서는 GIL 때문에 순수 Python 바이트코드 실행이 한 스레드에 제한된다. CPU-bound 작업은 프로세스나 GIL을 해제하는 네이티브 라이브러리를 우선 검토한다.

### “GIL이 있으므로 경쟁 상태가 없다”

GIL은 인터프리터 내부를 보호하는 잠금이지 애플리케이션의 논리적 작업을 보호하는 잠금이 아니다. 여러 단계로 구성된 공유 상태 변경에는 여전히 동기화가 필요하다.

### “`asyncio`는 여러 CPU 코어를 사용한다”

일반적인 `asyncio` 이벤트 루프는 한 스레드에서 코루틴을 번갈아 실행하는 동시성 모델이다. CPU 계산을 오래 수행하면 다른 코루틴도 실행되지 않으므로 계산은 프로세스나 별도 워커로 분리해야 한다.

### “Python이면 모든 라이브러리가 GIL의 영향을 똑같이 받는다”

Python 함수가 실행되는 구간과 C/C++/Rust 같은 네이티브 코드가 실행되는 구간은 다를 수 있다. 수치 연산, 압축, 해싱 라이브러리가 GIL을 해제하는지 문서와 벤치마크로 확인해야 한다.

## Python 3.13 이후의 변화

Python 3.13부터 `--disable-gil`로 빌드한 free-threaded CPython을 사용할 수 있다. 이 빌드에서는 여러 스레드가 Python 바이트코드를 동시에 실행할 수 있지만, 기본 배포 빌드의 실행 모델과 모든 라이브러리의 호환성이 자동으로 바뀌는 것은 아니다.

따라서 현재 코드를 이해할 때는 다음 순서가 안전하다.

1. 사용 중인 Python이 기본 GIL 빌드인지 확인한다.
2. 사용 중인 라이브러리가 free-threaded 빌드를 지원하는지 확인한다.
3. GIL을 끄면 공유 상태에 필요한 명시적 동기화가 더 중요해진다는 점을 고려한다.
4. 특정 방식이 빠르다고 가정하지 말고 동일한 입력으로 벤치마크한다.

GIL의 미래가 바뀌더라도 CPU-bound와 I/O-bound를 구분하고, 데이터 공유 비용과 동기화 비용을 측정하는 원칙은 그대로 유효하다.

## 정리

- GIL은 기본 CPython에서 한 번에 하나의 스레드만 Python 바이트코드를 실행하게 한다.
- `threading`은 순수 Python CPU 계산보다 I/O 대기 작업에 적합하다.
- CPU-bound 작업을 여러 코어에서 실행하려면 `multiprocessing`이나 `ProcessPoolExecutor`를 검토한다.
- `asyncio`는 `await` 지점에서 제어권을 넘기는 비동기 동시성 모델이다.
- 동기 블로킹 라이브러리는 `ThreadPoolExecutor`로 감싸는 방법을 고려할 수 있다.
- GIL은 경쟁 상태를 해결하지 않으므로 공유 가변 상태에는 별도 동기화가 필요하다.
- 프로세스, 스레드, 코루틴은 성능 특성과 비용이 다르므로 측정 결과를 기준으로 선택해야 한다.

## 학습 체크리스트

- [ ] 동시성과 병렬성의 차이를 설명할 수 있는가?
- [ ] 현재 작업이 CPU-bound인지 I/O-bound인지 판단할 수 있는가?
- [ ] GIL이 순수 Python 멀티스레딩의 CPU 성능을 제한하는 이유를 설명할 수 있는가?
- [ ] `asyncio` 코루틴 안에서 블로킹 호출이 왜 문제인지 설명할 수 있는가?
- [ ] `ProcessPoolExecutor`에서 프로세스 간 데이터 전달 비용을 고려했는가?
- [ ] 공유 가변 상태에 필요한 락이나 메시지 큐를 설계했는가?
- [ ] 실제 환경에서 단일 실행, 스레드, 프로세스의 시간을 측정했는가?

## 참고

- [Python Glossary: Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)
- [Python C API: Thread states and the global interpreter lock](https://docs.python.org/3/c-api/threads.html)
- [Python `threading` documentation](https://docs.python.org/3/library/threading.html)
- [Python `multiprocessing` documentation](https://docs.python.org/3/library/multiprocessing.html)
- [Python `asyncio` documentation](https://docs.python.org/3/library/asyncio.html)
- [Python `concurrent.futures` documentation](https://docs.python.org/3/library/concurrent.futures.html)
- [PEP 703: Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)
