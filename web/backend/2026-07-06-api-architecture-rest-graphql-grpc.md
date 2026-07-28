---
layout: post
title: "대규모 아키텍처의 API 설계: REST, GraphQL, gRPC 비교"
date: 2026-07-06
categories: [web, backend]
tags: [api, architecture, rest, graphql, grpc, microservices, distributed-system]
description: "RESTful API 설계 원칙을 이해하고 GraphQL과 gRPC의 통신 모델, 트레이드오프, 도메인 적합성을 대규모 아키텍처 관점에서 비교한다."
---

## 학습 목적

서비스 규모가 커지면 API는 단순히 데이터를 주고받는 함수가 아니다. 클라이언트와 서버의 배포 주기, 캐시와 관찰 가능성, 팀 간 계약, 브라우저와 모바일 지원, 서비스 간 통신량까지 결정하는 아키텍처 경계가 된다.

REST, GraphQL, gRPC는 모두 네트워크를 통해 데이터를 전달하지만 문제를 해결하는 방식은 다르다. 어떤 기술이 더 좋은지보다 다음 질문에 답하는 것이 중요하다.

- 외부 클라이언트와 통신하는가, 내부 서비스끼리 통신하는가?
- 클라이언트마다 필요한 데이터 형태가 크게 다른가?
- 사람이 직접 확인하고 디버깅해야 하는가?
- 낮은 지연 시간, 높은 처리량, 스트리밍이 중요한가?
- 캐시, 권한, 스키마 변경, 모니터링을 어떻게 운영할 것인가?

## 먼저 이해할 것: API와 프로토콜은 다르다

API는 컴포넌트가 어떤 기능과 데이터를 제공하는지 정한 계약이다. 프로토콜과 직렬화 형식은 그 계약을 네트워크에서 전달하는 방법이다.

```text
API 계약       무엇을 요청하고 어떤 결과를 받는가
전송 프로토콜  HTTP/1.1, HTTP/2 등 어떻게 전달하는가
직렬화 형식    JSON, Protocol Buffers 등 데이터를 어떻게 표현하는가
```

REST는 특정 라이브러리나 JSON 규격의 이름이 아니라 네트워크 기반 시스템을 위한 아키텍처 스타일이다. GraphQL은 클라이언트 요청 언어와 실행 모델이고, gRPC는 서비스 메서드를 호출하는 RPC 프레임워크다.

## RESTful API 설계 원칙

REST는 웹의 확장성과 독립적인 컴포넌트 배포를 높이기 위해 여러 아키텍처 제약을 조합한다. 실무에서 말하는 RESTful API는 이 원칙을 HTTP API에 적용한 형태다.

### 1. 리소스를 중심으로 설계한다

URL에는 동작보다 리소스를 표현하고, 동작은 HTTP 메서드로 구분한다.

```text
GET    /articles              게시글 목록 조회
GET    /articles/42           게시글 42 조회
POST   /articles              게시글 생성
PATCH  /articles/42           게시글 일부 수정
DELETE /articles/42           게시글 삭제
```

`/getArticles`, `/createArticle`처럼 URL에 동사를 반복하면 HTTP 메서드의 의미와 API의 자원 모델이 흐려진다. 다만 `POST /articles/42/publish`처럼 단순 CRUD로 표현하기 어려운 도메인 명령은 별도의 action 리소스로 설계할 수 있다.

### 2. HTTP 의미를 보존한다

HTTP 메서드와 상태 코드는 클라이언트, 프록시, 캐시, 모니터링 도구가 이해할 수 있는 공통 의미를 제공한다.

| 상황 | 예시 상태 코드 | 의미 |
| --- | ---: | --- |
| 조회·생성 성공 | `200`, `201` | 요청을 정상 처리함 |
| 비동기 작업 접수 | `202` | 처리 요청은 접수했지만 완료되지 않음 |
| 잘못된 요청 | `400` | 요청 형식이나 값이 잘못됨 |
| 인증 필요 | `401` | 인증 정보가 없거나 유효하지 않음 |
| 권한 부족 | `403` | 인증했지만 작업 권한이 없음 |
| 리소스 없음 | `404` | 대상 리소스를 찾을 수 없음 |
| 충돌 | `409` | 현재 상태와 요청이 충돌함 |
| 서버 오류 | `500` | 서버 내부 처리 실패 |

상태 코드를 항상 `200`으로 반환하고 본문 안에 별도 에러 코드를 넣으면 HTTP 기반 도구가 실패를 감지하기 어려워진다. 애플리케이션 에러 코드는 필요할 수 있지만 HTTP 상태 코드와 함께 사용해야 한다.

### 3. Stateless를 유지한다

각 요청은 처리에 필요한 정보를 스스로 포함해야 한다. 서버가 이전 요청의 실행 상태에 의존하지 않으면 서버 인스턴스를 수평 확장하고 요청을 여러 인스턴스에 분배하기 쉽다.

```text
클라이언트 -> 어느 API 인스턴스든 같은 요청 처리 가능
            -> 로컬 세션 메모리에 의존하지 않음
            -> 로드 밸런서와 오토스케일링 적용이 쉬움
```

Stateless가 로그인 상태나 데이터베이스 상태가 없다는 뜻은 아니다. 인증 토큰, 세션 저장소, DB 같은 상태를 요청 처리 서버의 로컬 메모리 밖에서 관리한다는 의미에 가깝다.

### 4. 캐시 가능성을 고려한다

GET 응답처럼 안전하게 재사용할 수 있는 데이터는 `Cache-Control`, `ETag`, `Last-Modified` 등을 사용해 캐시 정책을 설계할 수 있다. 캐시는 응답 속도와 서버 부하를 개선하지만, 사용자별 데이터가 섞이지 않도록 인증 정보와 캐시 키를 함께 검토해야 한다.

### 5. 계층 구조를 허용한다

클라이언트는 직접 애플리케이션 서버에 연결할 수도 있고 CDN, API Gateway, 로드 밸런서, 인증 프록시를 거쳐 연결할 수도 있다. 클라이언트가 중간 계층의 존재를 몰라도 되도록 경계를 설계하면 보안과 운영 정책을 중앙화할 수 있다.

### 6. 일관된 인터페이스를 만든다

리소스 표현, HTTP 메서드, 상태 코드, 에러 형식, 페이지네이션, 정렬과 필터링 규칙을 API 전체에서 일관되게 유지해야 한다.

```json
{
  "items": [
    { "id": 42, "title": "API 설계" }
  ],
  "next_cursor": "eyJpZCI6NDJ9"
}
```

페이지마다 `page`, `offset`, `cursor`를 제각각 사용하면 클라이언트와 운영 도구의 복잡도가 올라간다. 팀 단위로 API 규칙을 문서화하고 lint 또는 schema 검증으로 유지하는 것이 좋다.

## REST의 장점과 한계

### 잘 맞는 경우

- 웹 브라우저, 모바일, 외부 파트너처럼 다양한 클라이언트가 접근한다.
- 리소스와 CRUD 관계가 명확하다.
- HTTP 캐시, CDN, 프록시, 표준 모니터링 도구를 활용해야 한다.
- 사람이 curl이나 브라우저 개발자 도구로 요청을 확인해야 한다.
- API를 공개 문서와 SDK로 제공해야 한다.

### 주의할 점

- 화면 하나에 여러 리소스가 필요한 경우 여러 endpoint를 호출해야 할 수 있다.
- 서버가 정한 응답 구조 때문에 모바일이나 프론트엔드에서 over-fetching·under-fetching이 발생할 수 있다.
- endpoint와 응답 버전이 많아지면 변경 관리가 필요하다.
- REST 원칙을 지키지 않고 단순한 HTTP 기반 RPC로만 사용하면 일관된 의미와 캐시 이점을 잃을 수 있다.

REST는 모든 API를 완벽하게 표현해야 하는 규칙집이 아니라, 네트워크 시스템을 확장하고 독립적으로 진화시키기 위한 제약의 조합이다. 실제 설계에서는 도메인 명령과 운영 요구를 함께 고려해야 한다.

## GraphQL의 통신 모델

GraphQL은 타입이 있는 스키마를 서버가 제공하고, 클라이언트가 필요한 필드와 관계를 쿼리로 선택하는 방식이다. 대표적으로 `query`, `mutation`, `subscription` 작업을 사용한다.

```graphql
query ArticleDetail($id: ID!) {
  article(id: $id) {
    id
    title
    author {
      name
    }
    tags {
      name
    }
  }
}
```

서버는 스키마를 기준으로 쿼리를 검증하고, 요청한 형태에 맞춰 응답한다.

```json
{
  "data": {
    "article": {
      "id": "42",
      "title": "API 설계",
      "author": { "name": "A" },
      "tags": [{ "name": "backend" }]
    }
  }
}
```

### GraphQL이 해결하는 문제

- 화면마다 필요한 필드가 다른 경우 클라이언트가 응답 형태를 조절할 수 있다.
- 여러 리소스와 관계를 한 요청에서 표현할 수 있다.
- 강한 타입 스키마와 introspection으로 도구, 문서, 코드 생성을 연결할 수 있다.
- 모바일 네트워크처럼 요청 횟수와 응답 크기를 줄이는 것이 중요한 환경에 유리할 수 있다.

### GraphQL의 트레이드오프

- 클라이언트가 복잡한 쿼리를 만들 수 있으므로 depth, 비용, timeout, rate limit을 제한해야 한다.
- 쿼리 하나가 여러 resolver를 호출하므로 N+1, 데이터베이스 fan-out, 권한 검사를 별도로 관리해야 한다.
- 모든 요청이 같은 URL로 들어오면 HTTP 캐시와 CDN 캐시를 REST처럼 단순하게 적용하기 어렵다.
- 필드 단위 권한, 오류가 일부 데이터와 함께 반환되는 실행 모델, observability 설계가 필요하다.
- 스키마가 공개되면 필드 추가·deprecate·삭제와 호환성 정책을 장기적으로 관리해야 한다.

GraphQL은 endpoint 수를 줄이는 기술이지만 서버 내부의 데이터 조회 횟수까지 자동으로 줄여주는 기술은 아니다. resolver의 batching, DataLoader, ORM prefetch와 함께 설계해야 한다.

## gRPC의 통신 모델

gRPC는 원격의 서비스를 로컬 함수처럼 호출하는 RPC 프레임워크다. 서비스와 메시지를 Protocol Buffers로 정의하고, 그 계약에서 서버·클라이언트 코드를 생성한다.

```proto
syntax = "proto3";

service ArticleService {
  rpc GetArticle(GetArticleRequest) returns (Article);
}

message GetArticleRequest {
  string id = 1;
}

message Article {
  string id = 1;
  string title = 2;
}
```

일반적인 흐름은 다음과 같다.

```text
.proto 계약 작성
       |
       v
언어별 client/server stub 생성
       |
       v
클라이언트가 RPC 메서드 호출
       |
       v
HTTP/2 + Protocol Buffers로 전달
       |
       v
서버 구현 실행 및 응답 반환
```

gRPC는 단일 요청/응답뿐 아니라 서버 스트리밍, 클라이언트 스트리밍, 양방향 스트리밍을 지원한다. HTTP/2 기반 전송과 바이너리 직렬화, 코드 생성, deadline·cancellation·metadata·상태 코드 같은 기능을 함께 제공한다.

### gRPC가 잘 맞는 경우

- 조직 내부의 마이크로서비스 간 통신이 중심이다.
- 서비스 간 계약을 명확히 하고 여러 언어의 클라이언트 코드를 생성해야 한다.
- 낮은 지연 시간, 높은 처리량, 작은 메시지, 장시간 스트리밍이 중요하다.
- 브라우저가 아닌 백엔드 워커, 배치, 모바일 네이티브 앱이 주요 클라이언트다.
- 서비스 간 deadline, cancellation, health check, tracing을 공통 방식으로 운영해야 한다.

### gRPC의 트레이드오프

- Protocol Buffers와 stub 생성 도구를 빌드 파이프라인에 포함해야 한다.
- 바이너리 메시지는 JSON보다 사람이 직접 읽고 디버깅하기 어렵다.
- 일반 브라우저에서 native gRPC를 바로 사용하기 어렵고, gRPC-Web이나 gateway가 필요할 수 있다.
- HTTP/2, proxy, load balancer, timeout, keepalive, streaming 연결을 운영 환경에서 이해해야 한다.
- `.proto` 필드 번호와 호환성 규칙을 지키지 않으면 서비스 간 배포가 깨질 수 있다.
- 긴 스트림은 시작 후 일반적인 요청 단위 로드 밸런싱이나 장애 복구가 어려울 수 있다.

gRPC가 빠르다는 이유만으로 외부 공개 API에 사용하면 개발자 경험, 브라우저 호환성, 문서 접근성에서 비용이 커질 수 있다. 성능은 직렬화 형식 하나가 아니라 payload 크기, 연결 재사용, 네트워크 거리, 서버 처리, 호출 패턴을 함께 측정해야 한다.

## REST, GraphQL, gRPC 비교

| 기준 | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| 중심 모델 | 리소스와 HTTP | 타입 스키마와 클라이언트 쿼리 | 서비스 메서드와 메시지 |
| 대표 전송 | HTTP/JSON | 보통 HTTP/JSON | HTTP/2/Protobuf |
| 데이터 선택 | 서버가 endpoint별 정의 | 클라이언트가 필드 선택 | 계약에 정의된 메시지 |
| 브라우저 접근성 | 매우 높음 | 높음 | 별도 gRPC-Web/gateway 고려 |
| 캐시 | HTTP 캐시 활용이 쉬움 | 별도 캐시 정책 필요 | 애플리케이션·클라이언트 캐시 중심 |
| 타입·코드 생성 | OpenAPI 등 별도 도구 | 스키마·codegen 활용 | `.proto` 기반 강함 |
| 스트리밍 | 별도 SSE/WebSocket 등 조합 | Subscription 등 별도 운영 | 양방향 스트리밍 내장 |
| 디버깅 | curl·브라우저로 쉬움 | GraphiQL·쿼리 도구 필요 | 전문 도구·로그 변환 필요 |
| 주요 위험 | endpoint 증가, over-fetching | 쿼리 복잡도, resolver N+1 | 운영 복잡도, 브라우저·프록시 호환성 |

## 대규모 시스템에서의 조합

세 가지를 서비스 전체에서 하나만 선택할 필요는 없다. 클라이언트 경계와 서비스 내부 경계를 분리하면 각 통신의 요구에 맞는 방식을 사용할 수 있다.

```text
웹 브라우저 / 외부 파트너
          |
          v
   REST API 또는 GraphQL BFF
          |
          +------------------+
          |                  |
          v                  v
   Article Service      User Service
          |                  |
          +------ gRPC -------+
```

### REST + gRPC 조합

외부 공개 API는 REST로 제공하고, API Gateway 또는 BFF가 내부 서비스와 gRPC로 통신하는 구조다.

- 외부에는 HTTP 도구와 문서 생태계를 제공한다.
- 내부에는 강한 계약, 코드 생성, 효율적인 바이너리 통신을 사용한다.
- gateway가 인증, rate limit, 외부 응답 형식과 내부 서비스 계약을 분리한다.

### GraphQL + gRPC 조합

프론트엔드 요구가 다양하면 GraphQL BFF가 여러 내부 gRPC 서비스를 조합할 수 있다.

- 클라이언트는 화면 중심의 쿼리를 사용한다.
- BFF는 resolver에서 내부 서비스를 호출하고 응답을 조합한다.
- 내부 서비스는 `.proto` 계약과 gRPC를 사용한다.

이 경우 BFF가 단순 전달 계층이 아니라 fan-out, batching, 권한, timeout, 부분 실패를 책임지므로 운영 복잡도를 별도로 관리해야 한다.

## 선택 기준

```text
클라이언트가 누구인가?
├─ 브라우저/외부 파트너
│  ├─ 리소스와 HTTP 캐시가 중심인가? -> REST
│  └─ 화면별 데이터 조합과 필드 선택이 중요한가? -> GraphQL
└─ 내부 서비스/워커/네이티브 앱
   ├─ 강한 계약과 코드 생성이 중요한가? -> gRPC
   ├─ 스트리밍이 핵심인가? -> gRPC 우선 검토
   └─ 단순하고 공개적인 HTTP 접근이 필요한가? -> REST
```

| 요구사항 | 우선 검토 | 이유 |
| --- | --- | --- |
| 공개 API, 파트너 연동, 브라우저 중심 | REST | 접근성, HTTP 의미, 문서와 캐시 생태계 |
| 여러 화면과 클라이언트의 서로 다른 데이터 요구 | GraphQL | 클라이언트가 응답 필드와 관계를 선택 |
| 서비스 간 낮은 지연과 강한 계약 | gRPC | Protobuf 계약, stub 생성, 효율적 전송 |
| 실시간 양방향 데이터 | gRPC streaming 또는 WebSocket/SSE | 통신 방향과 클라이언트 지원을 함께 판단 |
| 초기 팀의 단순한 CRUD 서비스 | REST | 학습·운영·디버깅 비용이 낮음 |
| 여러 팀이 공유하는 복잡한 데이터 그래프 | GraphQL | 공통 타입 스키마와 집계 계층 구성 |

## 설계 전에 확인할 질문

- API의 주요 사용자는 브라우저, 모바일, 외부 파트너, 내부 서비스 중 누구인가?
- 요청 하나가 평균적으로 몇 개의 백엔드 서비스를 호출하게 되는가?
- 캐시해야 하는 데이터와 사용자별 데이터의 경계는 명확한가?
- API 오류, timeout, retry, idempotency, cancellation을 어떻게 정의할 것인가?
- 스키마 변경 시 구버전 클라이언트를 얼마나 오래 지원해야 하는가?
- 인증과 권한을 endpoint, field, RPC 메서드 중 어느 수준에서 검사할 것인가?
- 최대 payload, 쿼리 깊이, pagination, rate limit, deadline을 어떻게 제한할 것인가?
- 로그, trace, metric에서 한 사용자 요청이 내부 호출로 어떻게 이어지는지 볼 수 있는가?
- 실제 트래픽으로 p95/p99 지연, 처리량, CPU, 메모리, 네트워크 비용을 측정했는가?

## 정리

- REST는 리소스, HTTP 의미, Stateless, 캐시와 계층 구조를 활용하는 웹 중심 아키텍처 스타일이다.
- GraphQL은 강한 타입 스키마를 기반으로 클라이언트가 필요한 응답 형태를 선택하는 데 적합하다.
- gRPC는 계약 기반 서비스 호출, 코드 생성, 효율적인 바이너리 통신과 스트리밍에 강점이 있다.
- GraphQL은 endpoint 수를 줄여도 resolver의 N+1이나 쿼리 복잡도 문제가 자동으로 해결되지 않는다.
- gRPC는 내부 통신에 강하지만 브라우저 접근성, 디버깅, 프록시와 운영 도구를 함께 검토해야 한다.
- 대규모 시스템에서는 외부 REST/GraphQL과 내부 gRPC를 gateway 또는 BFF로 조합할 수 있다.
- 기술 선택은 유행이나 단순 벤치마크보다 클라이언트, 데이터 형태, 캐시, 운영 역량, 장애 모델을 기준으로 해야 한다.

## 학습 체크리스트

- [ ] REST와 RESTful HTTP API의 차이를 설명할 수 있는가?
- [ ] 리소스, HTTP 메서드, 상태 코드, Stateless, 캐시 정책을 일관되게 설계했는가?
- [ ] GraphQL 쿼리 복잡도와 resolver N+1을 제한할 방법이 있는가?
- [ ] gRPC `.proto` 필드 번호와 호환성 규칙을 지킬 수 있는가?
- [ ] 공개 경계와 내부 서비스 경계에 같은 통신 방식을 강제하고 있지는 않은가?
- [ ] timeout, retry, idempotency, tracing을 통신 방식별로 정의했는가?
- [ ] 실제 운영 트래픽에서 p95/p99와 자원 비용을 측정했는가?

## 참고

- [Roy Fielding: REST architectural style dissertation abstract](https://ics.uci.edu/~fielding/pubs/dissertation/abstract.htm)
- [GraphQL official site](https://graphql.org/)
- [GraphQL specification](https://spec.graphql.org/September2025/)
- [gRPC: What is gRPC?](https://grpc.io/docs/what-is-grpc/)
- [gRPC FAQ](https://grpc.io/docs/what-is-grpc/faq/)
- [gRPC performance best practices](https://grpc.io/docs/guides/performance/)
