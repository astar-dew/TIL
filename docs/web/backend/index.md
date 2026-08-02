# Web Backend

서버 애플리케이션과 API 개발에 필요한 내용을 정리합니다.

## Topics

- REST API
- 인증과 인가
- 서버 프레임워크
- 비즈니스 로직 설계
- 예외 처리
- 테스트
- 성능과 확장성

## Notes

- [Django ORM과 DRF의 N+1 문제 최적화](./2026-07-20-django-drf-internals-and-tradeoffs.md): N+1 발생 원인과 `select_related`, `prefetch_related`, 쿼리 검증 방법 정리
- [Django Middleware와 Signal의 동작 방식 및 트레이드오프](./2026-07-10-django-middleware-signal.md): 요청·응답 처리 순서와 Signal의 transaction·유지보수 이슈 정리
- [대규모 아키텍처의 API 설계: REST, GraphQL, gRPC 비교](./2026-07-06-api-architecture-rest-graphql-grpc.md): API 통신 모델과 도메인별 선택 기준, 대규모 시스템 조합 방식 정리
- [Redis 캐시 전략과 데이터 구조 활용](./2026-07-03-redis-cache-strategy.md): Look-aside, Write-through, Redis 자료구조, Cache Stampede 해결책 정리
