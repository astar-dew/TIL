---
title: "모니터링 스택 비교: Prometheus, ELK, Loki, 그리고 무엇을 언제 쓸까"
date: 2026-08-06
tags: [monitoring, observability, prometheus, elk, elasticsearch, loki, grafana, opentelemetry, sre, alerting]
description: "메트릭·로그·트레이스라는 축으로 주요 모니터링 도구의 특징을 비교하고, 서비스 규모와 아키텍처에 따라 어떤 조합을 선택해야 하는지 정리한다."
---

## 학습 목적

"모니터링 뭐 쓰지?"에서 시작하면 답이 안 나온다. Prometheus와 ELK는 **경쟁 관계가 아니라 다루는 데이터 자체가 다르기** 때문이다. 둘 다 쓰는 회사가 많은 이유가 여기 있다.

그래서 도구 이름보다 **무엇을 관측하려는가**를 먼저 나눠야 한다. 이 글에서는 그 축을 정리한 뒤 주요 도구를 비교하고, 서비스 규모와 아키텍처에 따라 무엇을 고를지 정리한다.

## 먼저 나눠야 할 세 가지 데이터

관측 데이터는 성격이 다른 세 종류로 나뉜다. 흔히 관측 가능성(Observability)의 세 기둥이라 부른다.

| 구분 | 무엇인가 | 답하는 질문 | 데이터 특성 |
| --- | --- | --- | --- |
| **Metrics (메트릭)** | 시간에 따른 숫자 | "지금 정상인가?" "언제부터 나빠졌나?" | 작고 규칙적. 장기 보관 쉬움 |
| **Logs (로그)** | 개별 사건의 기록 | "그 요청에 정확히 무슨 일이 있었나?" | 크고 불규칙. 저장 비용 높음 |
| **Traces (트레이스)** | 요청 하나가 거쳐 간 경로 | "어느 서비스에서 느려졌나?" | 매우 큼. 보통 샘플링 |

각각의 대표 도구가 다르다.

```text
Metrics  →  Prometheus  (+ Grafana, Alertmanager)
Logs     →  ELK / OpenSearch, Loki
Traces   →  Jaeger, Tempo
계측 표준 →  OpenTelemetry
```

실무 흐름으로 보면 이렇게 이어진다.

```text
1. 메트릭이 이상을 알려준다        "5xx 비율이 3%로 올랐다"
       ↓
2. 트레이스가 위치를 좁힌다        "결제 서비스 응답이 느리다"
       ↓
3. 로그가 원인을 확정한다          "DB 커넥션 풀 고갈"
```

**메트릭으로 알아채고, 로그로 조사한다.** 이 순서를 알면 왜 둘 다 필요한지 이해된다.

## Prometheus

### 특징

시계열 메트릭에 특화된 오픈소스 도구다. CNCF 졸업 프로젝트이며 사실상 표준에 가깝다.

가장 큰 특징은 **Pull 방식**이다. 각 애플리케이션이 `/metrics` 엔드포인트를 열어두면 Prometheus가 주기적으로 긁어간다(scrape).

```text
Prometheus  ──HTTP GET /metrics──▶  애플리케이션
            ──HTTP GET /metrics──▶  node-exporter
            ──HTTP GET /metrics──▶  DB exporter
```

```text
# 애플리케이션이 노출하는 형태
http_requests_total{method="POST",path="/api/orders",status="500"} 27
http_request_duration_seconds_bucket{le="0.5"} 1832
```

**PromQL**로 질의한다.

```promql
# 최근 5분간 5xx 비율
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# 95 퍼센타일 응답 시간
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

알림은 별도 컴포넌트인 **Alertmanager**가 담당한다. 중복 제거, 그룹화, 무음 처리, Slack·PagerDuty 연동을 여기서 한다.

### 강점과 한계

| 강점 | 한계 |
| --- | --- |
| 설치와 운영이 단순하다 | 기본 구성이 단일 노드다 |
| Kubernetes와 궁합이 매우 좋다 | 장기 보존이 약하다 (로컬 디스크) |
| exporter 생태계가 풍부하다 | **고카디널리티에 취약하다** |
| PromQL이 강력하다 | 로그·트레이스는 다루지 못한다 |
| 대상이 죽으면 그 자체로 감지된다 | 서버리스처럼 pull이 불가능한 환경에 약하다 |

장기 보존과 고가용성이 필요해지면 **Thanos**나 **Grafana Mimir** 같은 도구를 얹어 오브젝트 스토리지에 저장한다.

### 어떤 상황에 적합한가

- **Kubernetes 환경.** 서비스 디스커버리로 파드를 자동 감지한다. 사실상 기본 선택지다.
- **인프라·애플리케이션 지표 감시.** CPU, 메모리, 요청 수, 응답 시간, 큐 길이.
- **알림이 필요한 모든 서비스.** 임계치 기반 알림의 표준 구성이다.
- **직접 운영할 수 있는 팀.** 관리형이 필요하면 Grafana Cloud나 AWS AMP를 쓴다.

## ELK / Elastic Stack

### 구성

로그를 **전문 검색(full-text search)** 으로 다루는 스택이다.

| 구성 요소 | 역할 |
| --- | --- |
| **Elasticsearch** | 저장과 검색 엔진 |
| **Logstash** | 수집·변환·전송 (무겁다) |
| **Kibana** | 시각화와 검색 UI |
| **Beats / Fluent Bit** | 경량 수집기 |

요즘은 무거운 Logstash 대신 **Filebeat**나 **Fluent Bit**로 수집하고, 필요할 때만 Logstash로 변환하는 구성이 많다. Kubernetes에서는 Fluentd를 쓴 **EFK** 조합도 흔하다.

### 강점과 한계

| 강점 | 한계 |
| --- | --- |
| **전문 검색이 매우 강력하다** | 리소스를 많이 먹는다 (JVM, 메모리) |
| 임의의 필드로 집계·필터가 가능하다 | 운영 난이도가 높다 (샤드, 인덱스 수명 관리) |
| Kibana 대시보드가 풍부하다 | 저장 비용이 크다 (모든 필드를 색인) |
| 로그 외 검색 용도로도 쓸 수 있다 | 클러스터 튜닝 지식이 필요하다 |

"사용자 이메일이 포함된 에러 로그를 지난 30일에서 찾아줘" 같은 **임의 조건 검색**은 ELK의 영역이다. Prometheus로는 불가능하다.

### 라이선스와 OpenSearch

2021년 Elastic이 라이선스를 변경하면서 AWS가 **OpenSearch**로 분기했다. 이후 Elastic이 AGPL 옵션을 다시 추가했지만, 두 갈래는 각자 발전하고 있다. AWS 환경이라면 관리형 OpenSearch Service가 자연스러운 선택이 된다.

### 어떤 상황에 적합한가

- **로그를 자주, 복잡하게 검색해야 하는 서비스.** 고객 문의 대응, 장애 조사.
- **감사(audit) 로그 보관이 요구되는 도메인.** 금융, 의료, 커머스 결제.
- **로그에서 지표를 뽑아야 할 때.** 예를 들어 특정 문구가 등장한 횟수 추이.
- **운영 인력이 있는 조직.** 소규모 팀이 직접 운영하기에는 부담이 크다.

## Grafana Loki

### 특징

"로그를 위한 Prometheus"를 표방한다. 결정적인 차이는 **로그 본문을 색인하지 않는다**는 점이다.

```text
Elasticsearch : 모든 필드를 색인 → 검색 빠름, 저장 비용 큼
Loki          : 라벨만 색인, 본문은 압축 저장 → 저장 저렴, 검색은 스캔 방식
```

라벨은 Prometheus와 같은 개념이다.

```logql
# LogQL — PromQL과 문법이 비슷하다
{app="api", env="prod"} |= "error" | json | status >= 500
```

### 강점과 한계

| 강점 | 한계 |
| --- | --- |
| 저장 비용이 매우 저렴하다 | 전문 검색 성능은 ELK에 못 미친다 |
| Grafana에서 메트릭과 함께 본다 | 넓은 시간 범위 검색이 느릴 수 있다 |
| Prometheus와 라벨 체계를 공유한다 | 라벨 설계를 잘못하면 성능이 급락한다 |
| 운영이 상대적으로 단순하다 | 복잡한 집계 분석에는 부적합 |

**메트릭에서 이상을 발견하고 같은 라벨로 로그를 바로 여는 흐름**이 Grafana 안에서 매끄럽게 이어진다는 것이 가장 큰 실용적 장점이다.

### 어떤 상황에 적합한가

- **이미 Prometheus + Grafana를 쓰고 있는 팀.** 추가 학습 비용이 가장 적다.
- **로그 비용이 부담되는 서비스.** ELK 대비 저장 비용이 크게 낮다.
- **검색이 "특정 서비스의 최근 로그 훑어보기" 수준인 경우.**
- **소규모~중규모 팀.**

## 분산 추적: Jaeger, Tempo, OpenTelemetry

마이크로서비스에서 "느리다"는 신고가 들어왔을 때, 어느 서비스가 원인인지 찾는 도구다.

```text
요청 하나의 여정 (Trace)
├─ API Gateway        12ms
├─ Auth Service        8ms
├─ Order Service     450ms  ← 여기
│  └─ DB Query       430ms  ← 실제 원인
└─ Notification       15ms
```

| 도구 | 특징 |
| --- | --- |
| **Jaeger** | CNCF 프로젝트. 추적 백엔드로 널리 쓰인다 |
| **Grafana Tempo** | 오브젝트 스토리지 기반으로 저렴. Grafana 통합 |
| **OpenTelemetry** | 도구가 아니라 **계측 표준** |

**OpenTelemetry(OTel)는 따로 알아둘 가치가 있다.** 메트릭·로그·트레이스를 수집하는 표준 규격과 SDK, 그리고 Collector를 제공한다. OTel로 계측해 두면 백엔드를 Jaeger에서 Tempo로, 또는 Datadog으로 바꿀 때 **애플리케이션 코드를 고치지 않아도 된다.** 벤더 종속을 피하는 가장 실질적인 방법이다.

트레이스는 데이터가 매우 크기 때문에 보통 **샘플링**한다. 전체의 1~10%만 저장하거나, 에러가 난 요청은 전부 저장하는 식(tail sampling)으로 구성한다.

## 상용 SaaS

직접 운영하지 않는 선택지다.

| 서비스 | 특징 | 비용 구조 |
| --- | --- | --- |
| **Datadog** | 메트릭·로그·트레이스·APM 통합. 가장 포괄적 | 호스트당 + 로그 GB당. **비용이 빠르게 증가** |
| **New Relic** | APM 강점 | 데이터 사용량 + 사용자 수 |
| **Grafana Cloud** | Prometheus/Loki/Tempo 관리형 | 메트릭 시리즈 수, 로그 GB |
| **Sentry** | **에러 추적 특화**. 스택 트레이스, 릴리스 추적 | 이벤트 수 |
| **CloudWatch** | AWS 네이티브 | 지표·로그 사용량 |

SaaS의 장점은 명확하다. **모니터링 시스템 자체를 운영하지 않아도 된다.** 모니터링 스택이 장애 나면 장애를 못 보는 아이러니도 피할 수 있다.

단점도 명확하다. **비용이 예측 불가능하게 늘어난다.** 로그 양이 늘거나 컨테이너 수가 늘면 청구서가 급증한다. 도입 시점에 "로그를 얼마나 보낼지"를 통제하는 규칙을 함께 정해야 한다.

**Sentry는 성격이 조금 다르다.** 범용 모니터링이 아니라 애플리케이션 에러에 특화되어 있어, 에러를 자동으로 그룹화하고 발생 추이와 영향 사용자 수를 보여준다. 다른 스택과 경쟁하지 않고 함께 쓰는 경우가 많다.

## 전체 비교표

| 항목 | Prometheus | ELK | Loki | Jaeger/Tempo | Datadog |
| --- | --- | --- | --- | --- | --- |
| 주 데이터 | 메트릭 | 로그 | 로그 | 트레이스 | 전부 |
| 수집 방식 | Pull | Push | Push | Push | Push (에이전트) |
| 쿼리 | PromQL | Query DSL / KQL | LogQL | - | 자체 |
| 저장 비용 | 낮음 | **높음** | 낮음 | 중간 | 사용량 과금 |
| 운영 부담 | 낮음~중간 | **높음** | 낮음 | 중간 | **없음** |
| 검색 능력 | 해당 없음 | **매우 강함** | 보통 | - | 강함 |
| 학습 곡선 | 중간 | 높음 | 낮음(Prom 경험 시) | 중간 | 낮음 |
| 알림 | Alertmanager | Kibana / Watcher | Grafana | - | 내장 |

## 어떤 상황에 무엇을 쓸까

### 서비스 규모·단계별

| 단계 | 권장 구성 | 이유 |
| --- | --- | --- |
| **개인 프로젝트 / 사이드** | 클라우드 기본 대시보드 + Sentry + Slack 알림 | 별도 운영 없이 에러 인지만 확보 |
| **초기 스타트업 (~5명)** | 관리형 SaaS 또는 Grafana Cloud 무료 티어 | 인프라 운영에 쓸 인력이 없다 |
| **성장기 (10~50명)** | Prometheus + Grafana + Loki | 비용과 통제력의 균형 |
| **대규모 / 여러 팀** | Prometheus(+Thanos) + ELK + OTel + 트레이싱 | 팀별 요구가 갈라진다 |
| **규제 산업** | ELK/OpenSearch 중심 + 장기 보관 | 감사 로그 검색이 필수 |

### 아키텍처별

| 아키텍처 | 특히 필요한 것 | 비고 |
| --- | --- | --- |
| **모놀리식** | 메트릭 + 로그 | 트레이싱 필요성이 낮다 |
| **마이크로서비스** | **트레이싱 필수** + 메트릭 + 로그 | 서비스 경계를 넘는 지연 추적 |
| **Kubernetes** | Prometheus 계열 | 서비스 디스커버리 궁합 |
| **서버리스 (Lambda, Workers)** | **Push 기반 / SaaS** | Pull scrape이 불가능하다 |
| **모바일 앱 백엔드** | 에러 추적(Sentry) + 메트릭 | 클라이언트 오류도 함께 봐야 한다 |

### 서버리스는 별도로 봐야 한다

Prometheus의 pull 방식은 **항상 떠 있는 대상**을 전제한다. Cloudflare Workers나 AWS Lambda처럼 요청이 있을 때만 실행되고 사라지는 환경에서는 긁어갈 대상이 없다.

선택지는 다음과 같다.

- 플랫폼 기본 관측 도구 사용 (CloudWatch, Workers Analytics Engine, Logpush)
- 애플리케이션에서 SaaS로 직접 push
- Prometheus **remote write**를 지원하는 에이전트로 push
- 배치 작업이라면 Pushgateway (단, 일반 서비스용이 아니다)

여기에 더해, 에러를 즉시 인지하는 최소 장치로 Slack 알림을 붙이는 방법은 [Cloudflare Workers 에러를 Slack으로 받기](./2026-08-05-slack-webhook-error-alert-workers.md)에 정리했다.

## 실전 조합 패턴

혼자 쓰는 경우는 드물고 보통 조합한다.

```text
[PLG 스택] 가장 흔한 자체 운영 조합
Prometheus (메트릭) + Loki (로그) + Grafana (통합 화면) + Alertmanager (알림)
→ 중소 규모, 비용 효율, Kubernetes

[Prometheus + ELK]
Prometheus (메트릭·알림) + ELK (로그 검색)
→ 로그 검색 요구가 강한 조직

[SaaS 올인원]
Datadog 또는 New Relic
→ 인력이 적고 예산은 있는 팀

[OTel 중심]
OpenTelemetry로 계측 → Collector → 백엔드(무엇이든)
→ 벤더 종속을 피하고 싶을 때. 신규 프로젝트라면 우선 검토할 만하다
```

## 더 알아두면 좋은 것

### 1. Push와 Pull의 차이

| | Pull (Prometheus) | Push (대부분) |
| --- | --- | --- |
| 대상 상태 확인 | **자연히 됨** (긁히지 않으면 죽은 것) | 별도 헬스체크 필요 |
| 방화벽 | 모니터링 서버가 대상에 접근해야 함 | 대상이 밖으로 나가면 됨 |
| 단명 작업 | **어려움** | 쉬움 |
| 부하 제어 | 수집 측이 조절 | 대상이 몰아칠 수 있음 |

### 2. 카디널리티 폭발

Prometheus를 쓰다 메모리가 터지는 가장 흔한 원인이다. 시계열은 **라벨 값 조합마다 하나씩** 생긴다.

```promql
# 위험 — user_id가 100만 명이면 시계열 100만 개
http_requests_total{user_id="12345", path="/api/orders"}

# 안전 — path 종류만큼만 생성
http_requests_total{path="/api/orders", status="200"}
```

**라벨에는 값의 종류가 적은 것만 넣는다.** 사용자 ID, 요청 ID, 이메일, 타임스탬프는 라벨이 아니라 **로그나 트레이스**에 담아야 한다. 이 구분이 세 기둥을 나누는 실용적 이유이기도 하다.

### 3. 무엇을 감시할지: 4 Golden Signals

Google SRE가 제시한 네 가지다. 무엇부터 계측할지 막막할 때 기준이 된다.

| 신호 | 의미 |
| --- | --- |
| **Latency** | 응답 시간. 성공 요청과 실패 요청을 나눠서 본다 |
| **Traffic** | 요청량 |
| **Errors** | 실패 비율 |
| **Saturation** | 자원 포화도 (가장 부족한 자원 기준) |

관점에 따라 두 가지 방법론도 함께 알아둘 만하다.

- **RED** (서비스 관점): Rate, Errors, Duration
- **USE** (자원 관점): Utilization, Saturation, Errors

### 4. 알림은 증상 기준으로

```text
나쁜 알림: "CPU 사용률 80% 초과"
        → 사용자에게 아무 영향이 없을 수도 있다. 새벽에 깨울 이유가 없다.

좋은 알림: "5분간 5xx 비율 5% 초과"
        → 사용자가 실제로 겪고 있는 문제다.
```

**사람을 깨우는 알림은 사용자 영향이 있는 것만**으로 제한한다. 원인 지표(CPU, 메모리)는 대시보드에서 조사할 때 보면 된다.

알림이 많아지면 사람들이 무시하게 되고, 결국 없는 것과 같아진다. Slack 알림에서 다뤘던 알림 피로 문제가 여기서도 똑같이 적용된다.

### 5. SLI / SLO / 에러 버짓

임계치를 감으로 정하지 않는 방법이다.

| 용어 | 의미 | 예시 |
| --- | --- | --- |
| **SLI** | 측정하는 지표 | 성공 응답 비율 |
| **SLO** | 목표치 | 30일간 99.9% |
| **에러 버짓** | 허용되는 실패량 | 0.1% = 약 43분/월 |

에러 버짓이 남아 있으면 배포 속도를 유지하고, 빠르게 소진되면 안정화에 집중한다는 식으로 **개발 속도와 안정성을 조율하는 기준**이 된다.

### 6. 구조화 로깅과 상관관계 ID

로그를 문자열로 남기면 검색과 집계가 어렵다. **JSON으로 남기면** 도구가 필드를 이해할 수 있다.

```javascript
// 나쁨
console.log(`user ${userId} failed to pay: ${error.message}`);

// 좋음
console.log(JSON.stringify({
  level: "error",
  event: "payment_failed",
  user_id: userId,
  order_id: orderId,
  trace_id: traceId,        // 트레이스와 로그를 잇는 열쇠
  error: error.message,
}));
```

`trace_id`를 로그에 함께 남기면 **트레이스에서 로그로, 로그에서 트레이스로** 이동할 수 있다. 세 기둥을 실제로 연결하는 것이 이 상관관계 ID다.

### 7. 보존 기간과 비용

관측 데이터는 오래될수록 가치가 떨어진다. 단계를 나눈다.

| 기간 | 처리 |
| --- | --- |
| 최근 7~14일 | 원본 그대로, 빠른 조회 |
| 1~3개월 | 다운샘플링(메트릭), 저렴한 스토리지(로그) |
| 1년 이상 | 규제 목적만 콜드 스토리지 보관 |

**로그는 무조건 다 모으지 않는다.** DEBUG 로그를 운영에서 그대로 수집하면 비용의 대부분을 여기서 쓰게 된다. 로그 레벨과 샘플링 정책을 먼저 정한다.

### 8. 모니터링과 관측 가능성의 차이

| | 모니터링 | 관측 가능성 |
| --- | --- | --- |
| 전제 | **예상한 문제**를 감시 | 예상 못 한 문제도 조사 가능 |
| 질문 | "CPU가 임계치를 넘었나?" | "왜 이 사용자만 느린가?" |
| 방식 | 미리 정의한 대시보드·알림 | 임의 질의로 파고들기 |

시스템이 복잡해질수록 "미리 정의한 지표"만으로는 부족해진다. 고카디널리티 데이터를 임의로 조합해 질문할 수 있어야 한다는 것이 관측 가능성의 문제의식이다.

### 9. 모니터링 시스템 자체의 가용성

자체 호스팅의 함정이다. 인프라가 통째로 죽으면 **모니터링도 함께 죽어서 알림이 오지 않는다.** 최소한 다음 중 하나는 외부에 둔다.

- 외부 업타임 체크 (UptimeRobot, Better Stack 등)
- 알림 경로만이라도 SaaS로 분리
- Dead man's switch: "정상 신호가 끊기면 알림"을 거는 방식

## 도입 순서 제안

한 번에 다 하려 하면 실패한다. 이 순서가 무난하다.

1. **에러 인지부터.** Sentry나 Slack 알림으로 "터진 걸 아는" 상태를 먼저 만든다.
2. **기본 메트릭 4개.** 요청량, 에러율, 응답 시간, 자원 사용률.
3. **알림 규칙 정의.** 사용자 영향 기준으로 최소한만.
4. **로그 중앙화.** 서버에 SSH로 들어가지 않고 볼 수 있게.
5. **대시보드 정리.** 장애 시 볼 화면을 미리 만든다.
6. **트레이싱.** 서비스가 여러 개로 나뉘었을 때.
7. **SLO 도입.** 감이 아니라 숫자로 판단하게 될 때.

## 자주 하는 실수

### Prometheus와 ELK를 양자택일로 생각하기

다루는 데이터가 다르다. 보통 함께 쓴다.

### 라벨에 고카디널리티 값 넣기

`user_id`, `request_id`를 메트릭 라벨에 넣으면 Prometheus가 메모리 부족으로 죽는다.

### 모든 로그를 수집하기

DEBUG 로그까지 다 보내면 비용의 대부분이 여기서 나간다. 레벨과 샘플링을 먼저 정한다.

### 원인 지표로 알림 걸기

CPU 80%로 사람을 깨우면 대부분 헛수고다. 사용자 영향 기준으로 건다.

### 대시보드만 만들고 알림은 안 걸기

아무도 대시보드를 상시 보고 있지 않다.

### SaaS 비용을 계산하지 않고 도입

로그 GB 과금은 트래픽이 늘면 급증한다. 도입 전에 예상 사용량을 계산한다.

### 모니터링 스택을 감시 대상과 같은 인프라에 두기

함께 죽으면 알림이 오지 않는다.

## 정리

- 메트릭·로그·트레이스는 성격이 다른 데이터이며, 도구 선택은 여기서 출발한다.
- Prometheus는 메트릭과 알림의 표준이고, pull 방식이라 Kubernetes와 궁합이 좋지만 서버리스에는 맞지 않는다.
- ELK는 로그 전문 검색이 강력한 대신 운영과 저장 비용이 크다.
- Loki는 라벨만 색인해 저렴하지만 검색 능력은 ELK보다 약하다. Prometheus를 쓰고 있다면 자연스러운 짝이다.
- 마이크로서비스라면 트레이싱이 사실상 필수이고, OpenTelemetry로 계측하면 백엔드를 바꿔도 코드를 고치지 않아도 된다.
- SaaS는 운영 부담이 없는 대신 비용이 사용량에 비례해 급증할 수 있다.
- 메트릭 라벨에 고카디널리티 값을 넣으면 안 된다. 그런 데이터는 로그와 트레이스에 담는다.
- 알림은 원인이 아니라 사용자 영향(증상) 기준으로 건다.
- 관측 데이터는 보존 단계를 나누고, 로그는 레벨과 샘플링으로 양을 통제한다.
- 모니터링 시스템은 감시 대상과 운명을 같이하지 않도록 분리한다.

## 학습 체크리스트

- [ ] 메트릭·로그·트레이스가 각각 어떤 질문에 답하는지 구분할 수 있는가?
- [ ] Prometheus가 pull 방식이라 얻는 이점과 한계를 설명할 수 있는가?
- [ ] ELK와 Loki의 색인 방식 차이와 그로 인한 비용 차이를 아는가?
- [ ] 서버리스 환경에서 Prometheus scrape이 어려운 이유와 대안을 아는가?
- [ ] 카디널리티 폭발이 왜 생기고 무엇을 라벨에 넣으면 안 되는지 아는가?
- [ ] 4 Golden Signals를 담당 서비스에 적용해 볼 수 있는가?
- [ ] 지금 걸려 있는 알림이 증상 기준인지 원인 기준인지 판단할 수 있는가?
- [ ] `trace_id`로 로그와 트레이스를 잇는 방식을 이해했는가?
- [ ] 담당 서비스의 관측 데이터 보존 기간과 비용을 파악하고 있는가?
- [ ] 인프라가 통째로 죽었을 때 알림이 오는 경로가 있는가?

## 참고

- [Prometheus — Overview](https://prometheus.io/docs/introduction/overview/)
- [Prometheus — Querying basics (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Prometheus — Instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [Elastic — Elastic Stack 소개](https://www.elastic.co/guide/en/starting-with-the-elasticsearch-platform-and-its-solutions/current/index.html)
- [Grafana Loki — Overview](https://grafana.com/docs/loki/latest/get-started/overview/)
- [Grafana Tempo — Documentation](https://grafana.com/docs/tempo/latest/)
- [OpenTelemetry — What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html)
