# DevOps

개발, 배포, 운영 자동화와 관련된 내용을 정리합니다.

## Topics

- Git, GitHub
- CI/CD
- Docker
- Cloud
- 배포 자동화
- 모니터링
- 로그 관리

## Notes

- [모니터링 스택 비교: Prometheus, ELK, Loki, 그리고 무엇을 언제 쓸까](./2026-08-06-monitoring-stack-comparison.md): 메트릭·로그·트레이스 축으로 도구를 비교하고, 규모·아키텍처별 선택 기준과 카디널리티·알림 설계·SLO 정리
- [GitHub 머지 전략 4가지: Merge Commit, Squash, Rebase, Fast-forward](./2026-08-05-github-merge-strategies.md): 같은 상황에 네 방식을 적용해 히스토리 변화를 비교하고, 되돌리기 난이도와 상황별 선택 기준 정리
- [Cloudflare Workers 에러를 Slack으로 받기: Incoming Webhook 알림 구성](./2026-08-05-slack-webhook-error-alert-workers.md): 웹훅 발급과 시크릿 관리, `waitUntil` 기반 비동기 전송, KV 쿨다운으로 알림 피로 줄이기, 민감정보 마스킹 정리
- [Git의 내부 구조: Merkle Tree와 커밋 DAG](./2026-08-04-git-object-model-merkle-tree.md): blob·tree·commit 객체, 해시 전파, 커밋 히스토리가 DAG인 이유와 rebase·reflog에 미치는 영향 정리
- [CI/CD 파이프라인 설계 원칙과 GitHub Actions 기본 구조](./2026-08-03-cicd-pipeline-github-actions.md): 아티팩트 1회 빌드, 빠른 피드백, 최소 권한 원칙과 Workflow/Job/Step 구조, 캐시·아티팩트·OIDC 설정 정리
- [Cloudflare Pages와 Workers 배포 중 캐시로 인한 오류](./2026-07-23-cloudflare-pages-workers-cache-deploy-error.md): React 프론트엔드 배포 중 캐시로 발생한 간헐적 빌드/배포 오류 정리
