---
title: "CI/CD 파이프라인 설계 원칙과 GitHub Actions 기본 구조"
date: 2026-08-03
tags: [cicd, github-actions, devops, deploy, automation, pipeline, oidc]
description: "CI/CD 파이프라인을 설계할 때 지켜야 할 원칙을 정리하고, GitHub Actions의 워크플로 구조와 트리거, 캐시, 권한 설정 방법을 실제 예시로 정리한다."
---

## 학습 목적

CI/CD는 "push하면 자동으로 배포되는 것" 정도로 이해해도 동작은 한다. 하지만 파이프라인이 느려지거나, 같은 커밋인데 스테이징과 운영의 결과가 다르거나, 실패 원인을 찾는 데 시간이 오래 걸리기 시작하면 그때부터는 설계 문제가 된다.

이 글에서는 도구와 무관하게 통하는 **파이프라인 설계 원칙**을 먼저 정리하고, 그 원칙을 **GitHub Actions**에서 어떤 문법으로 구현하는지 이어서 정리한다.

## CI와 CD 구분하기

세 용어가 자주 섞여 쓰인다.

| 용어 | 의미 | 끝나는 지점 |
| --- | --- | --- |
| CI (Continuous Integration) | 변경을 자주 통합하고 자동으로 검증한다 | 빌드와 테스트 통과 |
| CD (Continuous Delivery) | 언제든 배포 가능한 상태로 아티팩트를 준비한다 | 배포 직전, 승인 대기 |
| CD (Continuous Deployment) | 검증을 통과하면 사람 승인 없이 자동 배포한다 | 운영 환경 반영 |

Delivery와 Deployment의 차이는 **사람의 승인 단계가 있는가**다. 결제나 금융처럼 되돌리기 어려운 도메인은 Delivery까지만 자동화하고 운영 반영은 승인 뒤에 하는 선택이 일반적이다.

CI의 핵심은 자동화 자체가 아니라 **통합 주기를 짧게 가져가는 것**이다. 브랜치를 오래 유지할수록 충돌과 회귀 위험이 커지므로, 작게 자주 합치고 그때마다 검증하는 흐름이 목적이다.

## 파이프라인 설계 원칙

### 1. 한 번 빌드하고, 같은 산출물을 배포한다

가장 중요한 원칙이다. 환경마다 다시 빌드하면 스테이징에서 검증한 것과 운영에 올라간 것이 **다른 산출물**이 된다.

```text
잘못된 구조
  코드 ─▶ 스테이징용 빌드 ─▶ 스테이징
  코드 ─▶ 운영용 빌드     ─▶ 운영     (서로 다른 산출물)

권장 구조
  코드 ─▶ 빌드 1회 ─▶ 아티팩트 ─┬─▶ 스테이징
                              └─▶ 운영    (동일한 산출물)
```

환경별로 달라지는 값은 빌드에 넣지 말고 **실행 시점의 환경 변수나 설정**으로 주입한다. 프론트엔드처럼 빌드 시점에 값이 박히는 경우에는 환경별 빌드가 불가피할 수 있는데, 그때도 어떤 커밋에서 어떤 설정으로 만든 산출물인지 추적할 수 있어야 한다.

### 2. 빠른 피드백을 우선한다

개발자가 결과를 기다리는 시간이 길수록 파이프라인은 무시당한다. **싸고 빠른 검증을 앞에 배치**해 일찍 실패시킨다.

```text
린트·타입 검사 (수십 초)
   ↓ 통과
단위 테스트 (수 분)
   ↓ 통과
빌드 + 통합 테스트 (수 분~십수 분)
   ↓ 통과
E2E 테스트 (십 분 이상)
   ↓ 통과
배포
```

문법 오류 하나 때문에 E2E까지 다 돌고 20분 뒤에 실패를 알게 되면 안 된다. 서로 의존하지 않는 단계는 병렬로 실행해 전체 시간을 줄인다.

### 3. 재현 가능해야 한다

"내 로컬에서는 되는데"와 마찬가지로 "어제는 통과했는데"도 문제다.

- 의존성은 잠금 파일(`package-lock.json`, `poetry.lock`, `requirements.txt`의 버전 고정)로 정확한 버전을 고정한다.
- 런타임 버전(Node, Python, JDK)을 워크플로에 명시한다. `latest`에 의존하지 않는다.
- 서드파티 액션은 태그가 아니라 커밋 SHA로 고정하는 것이 가장 안전하다.
- 빌드가 시스템에 미리 설치된 도구에 의존하지 않게 한다.

### 4. 파이프라인 실패는 즉시 고친다

빨간 파이프라인을 방치하면 아무도 결과를 보지 않게 되고, CI는 형식만 남는다. 특히 **간헐적으로 실패하는 테스트(flaky test)** 는 신뢰를 가장 빠르게 무너뜨린다. 재실행으로 넘기지 말고 격리하거나 고쳐야 한다.

### 5. 최소 권한으로 실행한다

CI 러너는 저장소 코드와 배포 자격 증명을 동시에 다루므로 공격 대상이 되기 쉽다.

- 워크플로 토큰 권한은 기본을 읽기로 두고 필요한 job에만 쓰기를 준다.
- 장기 액세스 키 대신 **OIDC 기반 단기 자격 증명**을 사용한다.
- 시크릿을 로그에 출력하지 않는다.
- 외부 기여자의 PR이 시크릿에 접근하지 못하게 한다.

### 6. 되돌릴 수 있어야 한다

배포보다 **롤백이 더 빨라야** 한다. 이전 아티팩트를 보관하고, 어떤 커밋이 배포되었는지 추적할 수 있어야 하며, 문제가 생겼을 때 즉시 되돌릴 경로가 있어야 한다. 무중단 배포가 필요하면 블루-그린이나 카나리 방식을 검토한다.

### 7. 실패 원인이 드러나야 한다

실패했을 때 로그만 보고 원인을 알 수 있어야 한다. 테스트 리포트, 스크린샷, 빌드 로그를 아티팩트로 남기고, 실패 알림에 어떤 단계가 왜 실패했는지 포함한다.

## GitHub Actions 기본 구조

### 계층 구조

```text
Workflow (.github/workflows/deploy.yml)
├── Trigger (on)              언제 실행할지
└── Job                       하나의 러너(가상 머신)에서 실행되는 단위
    ├── runs-on               실행 환경
    └── Step                  순차 실행되는 명령
        ├── uses              재사용 가능한 Action 호출
        └── run               셸 명령 실행
```

- **Workflow**: `.github/workflows/` 아래 YAML 파일 하나가 워크플로 하나다.
- **Job**: 기본적으로 서로 **병렬 실행**되며, 각각 독립된 러너에서 돌아간다. 따라서 job 사이에는 파일이 자동으로 전달되지 않는다.
- **Step**: 같은 job 안에서 **순차 실행**되고 작업 디렉터리와 파일을 공유한다.
- **Action**: 재사용 가능한 단위이며 `uses`로 호출한다.

Job이 별도 러너에서 돈다는 점이 중요하다. 빌드 job의 결과를 배포 job에 넘기려면 아티팩트로 업로드·다운로드해야 한다.

### 최소 예시

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm test
```

`npm install`이 아니라 `npm ci`를 쓰는 이유는 잠금 파일 그대로 설치해 재현성을 지키기 위해서다.

### 주요 키워드

| 키워드 | 위치 | 역할 |
| --- | --- | --- |
| `on` | workflow | 실행 트리거 |
| `jobs` | workflow | job 정의 |
| `runs-on` | job | 러너 환경 (`ubuntu-latest`, `macos-latest` 등) |
| `needs` | job | 선행 job 지정 (의존 관계) |
| `if` | job/step | 조건부 실행 |
| `steps` | job | 실행할 단계 목록 |
| `uses` | step | Action 호출 |
| `run` | step | 셸 명령 실행 |
| `with` | step | Action에 전달할 입력 |
| `env` | 모든 레벨 | 환경 변수 |
| `permissions` | workflow/job | `GITHUB_TOKEN` 권한 |
| `concurrency` | workflow/job | 동시 실행 제어 |

## 트리거 설계

```yaml
on:
  push:
    branches: [main]
    paths:                    # 문서만 바뀌면 실행하지 않는다
      - "src/**"
      - "package.json"
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 18 * * *"      # UTC 기준. KST는 +9시간
  workflow_dispatch:          # 수동 실행 버튼 활성화
    inputs:
      environment:
        type: choice
        options: [staging, production]
```

| 트리거 | 용도 |
| --- | --- |
| `push` | 브랜치 반영 시 검증·배포 |
| `pull_request` | 머지 전 검증 |
| `workflow_dispatch` | 수동 실행 (배포, 롤백에 유용) |
| `schedule` | 정기 작업 (의존성 점검, 야간 테스트) |
| `release` | 릴리스 태그 기준 배포 |
| `workflow_call` | 다른 워크플로에서 재사용 |

`paths` 필터로 관련 없는 변경에 파이프라인이 도는 것을 막으면 대기 시간과 비용이 줄어든다. 단, **필수 상태 검사(required check)로 지정한 워크플로에 `paths`를 걸면 실행되지 않아 PR이 머지 대기 상태로 멈출 수 있으므로** 주의한다.

## Job 구성

### 의존 관계와 병렬 실행

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: [lint, test]        # 둘 다 통과해야 실행
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'   # main에서만 배포
    runs-on: ubuntu-latest
    steps: [...]
```

`lint`와 `test`는 동시에 실행되고, `build`는 둘 다 끝난 뒤 실행된다.

### 매트릭스로 여러 조합 검증

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # 하나 실패해도 나머지는 계속
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: ["18", "20", "22"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

2 × 3 = 6개 job이 병렬 실행된다. `fail-fast: false`로 두면 하나가 실패해도 나머지 결과를 모두 확인할 수 있어, 특정 버전에서만 나는 문제를 찾기 좋다.

### Job 간 값 전달

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - id: meta
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "배포 버전 ${{ needs.build.outputs.version }}"
```

## 캐시와 아티팩트

둘은 목적이 다르다.

| 구분 | 목적 | 수명 | 실패 시 |
| --- | --- | --- | --- |
| 캐시 | 의존성 재설치 시간 단축 | 자동 만료 | 없어도 동작해야 한다 |
| 아티팩트 | job 간 파일 전달, 결과물 보관 | 지정한 보관 기간 | 없으면 배포 불가 |

```yaml
# 캐시: 잠금 파일 해시가 키
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# 아티팩트: 빌드 job -> 배포 job 전달
- uses: actions/upload-artifact@v4
  with:
    name: dist
    path: dist/
    retention-days: 7

- uses: actions/download-artifact@v4
  with:
    name: dist
    path: dist/
```

`setup-node`, `setup-python`, `setup-java` 등은 `cache` 옵션을 내장하고 있어 대부분의 경우 `actions/cache`를 직접 쓸 필요가 없다.

**캐시는 정확성에 영향을 주면 안 된다.** 키에 잠금 파일 해시를 넣어 의존성이 바뀌면 키도 바뀌게 해야 오래된 캐시가 재사용되지 않는다.

## 권한과 시크릿

### 최소 권한 설정

`GITHUB_TOKEN`은 job마다 자동으로 발급되는 임시 토큰이다. 필요한 권한만 명시한다.

```yaml
permissions:
  contents: read              # 기본은 읽기만

jobs:
  deploy:
    permissions:
      contents: read
      id-token: write         # OIDC 사용 시 필요
      pages: write            # 이 job에만 부여
```

### OIDC로 장기 키 없애기

클라우드 자격 증명을 시크릿에 넣어 두면 유출 시 피해가 크다. OIDC를 쓰면 워크플로가 실행될 때마다 단기 토큰을 발급받는다.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions
      aws-region: ap-northeast-2
```

클라우드 쪽에서 신뢰 정책에 **저장소와 브랜치를 조건으로 명시**해야 한다. 조건 없이 열어 두면 다른 저장소에서도 그 역할을 가져갈 수 있다.

### 환경 보호 규칙

승인 단계가 필요하면 `environment`를 사용한다.

```yaml
jobs:
  deploy-production:
    environment:
      name: production        # 저장소 설정에서 승인자·대기 시간 지정
      url: https://example.com
    runs-on: ubuntu-latest
```

환경별로 시크릿을 분리할 수 있고, 지정한 리뷰어가 승인해야 job이 진행된다. Continuous Delivery 구조를 만들 때 쓰는 기능이다.

### `pull_request_target` 주의

`pull_request`는 포크에서 온 PR에 대해 시크릿 없이, 읽기 권한으로 실행된다. 반면 `pull_request_target`은 **베이스 저장소 컨텍스트에서 시크릿과 함께** 실행된다.

여기서 PR의 코드를 체크아웃해 실행하면 외부인이 보낸 코드가 시크릿에 접근하게 된다. 꼭 필요하다면 PR 코드를 실행하지 않는 범위(라벨링, 코멘트)로만 사용한다.

## 동시 실행 제어

배포 워크플로가 겹쳐 실행되면 나중 것이 먼저 끝나 오래된 버전이 배포될 수 있다.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true    # PR 검증: 이전 실행 취소
```

배포에서는 취소가 위험할 수 있으므로 `cancel-in-progress: false`로 두고 순서대로 실행되게 한다.

## 실제 예시: 이 저장소의 배포 워크플로

이 TIL 저장소는 `.github/workflows/deploy.yml`에서 MkDocs 사이트를 빌드해 GitHub Pages로 배포한다. 앞의 원칙이 어떻게 적용되는지 보기 좋은 예다.

```yaml
name: Deploy site

on:
  push:
    branches: [main]
  workflow_dispatch:            # 수동 재배포 경로 확보

permissions:                    # 최소 권한
  contents: read
  pages: write
  id-token: write               # OIDC

concurrency:
  group: pages
  cancel-in-progress: false     # 배포는 취소하지 않고 순서대로

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"   # 버전 고정
          cache: pip               # 내장 캐시
      - run: pip install -r requirements.txt
      - run: mkdocs build --strict # 경고를 실패로 처리
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: site               # 아티팩트로 전달

  deploy:
    needs: build                   # 빌드 성공 후에만
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

- `--strict`는 끊어진 링크 같은 경고를 실패로 처리해 잘못된 문서가 배포되지 않게 한다.
- 빌드와 배포를 job으로 분리해, 빌드 실패 시 배포가 아예 시작되지 않는다.
- 아티팩트를 한 번만 만들고 그것을 배포한다.

## 자주 하는 실수

### `npm install` 사용

잠금 파일과 다른 버전이 설치될 수 있다. CI에서는 `npm ci`, `pip install -r requirements.txt`(버전 고정), `poetry install --no-root` 등 재현 가능한 명령을 쓴다.

### 서드파티 액션을 태그로 참조

`uses: some/action@v1`에서 `v1` 태그는 언제든 다른 커밋을 가리킬 수 있다. 액션 저장소가 탈취되면 시크릿이 유출된다. 신뢰도가 낮은 액션일수록 커밋 SHA로 고정한다.

```yaml
- uses: some/action@a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0  # v1.2.3
```

### job 사이에 파일이 남아 있다고 가정

job은 각각 다른 러너에서 실행된다. 빌드 산출물은 아티팩트로 전달해야 한다.

### 배포 job에 조건이 없음

`needs`만 걸고 브랜치 조건을 빼면 PR에서도 배포가 실행될 수 있다. `if: github.ref == 'refs/heads/main'`처럼 명시한다.

### 시크릿을 로그에 노출

GitHub이 시크릿 값을 마스킹하지만, base64로 인코딩하거나 JSON에 담아 출력하면 마스킹을 우회할 수 있다. 애초에 출력하지 않는 것이 안전하다.

### 캐시를 정확성에 의존

캐시가 없어도 파이프라인은 정상 동작해야 한다. 빌드 산출물을 캐시로 넘기면 오래된 결과가 배포될 수 있다.

## 정리

- CI는 통합 주기를 짧게 유지하는 것이 목적이고, Delivery와 Deployment는 사람의 승인 단계 유무로 나뉜다.
- 한 번 빌드한 아티팩트를 모든 환경에 배포해야 검증한 것과 배포된 것이 같아진다.
- 싸고 빠른 검증을 앞에 두고, 독립적인 단계는 병렬로 실행해 피드백을 앞당긴다.
- 잠금 파일과 런타임 버전 고정으로 재현성을 확보한다.
- GitHub Actions는 Workflow > Job > Step 구조이며, job은 병렬·독립 러너, step은 순차·공유 환경이다.
- job 간 파일 전달은 아티팩트로 하고, 캐시는 속도만 담당하게 한다.
- `permissions`는 최소로 두고, 장기 키 대신 OIDC를 사용한다.
- 배포 워크플로는 `concurrency`로 순서를 지키고, 롤백 경로를 미리 준비한다.

## 학습 체크리스트

- [ ] Continuous Delivery와 Continuous Deployment의 차이를 설명할 수 있는가?
- [ ] 환경마다 재빌드하면 왜 위험한지 설명할 수 있는가?
- [ ] 파이프라인 단계를 어떤 기준으로 배치해야 피드백이 빨라지는가?
- [ ] Job과 Step의 실행 방식(병렬/순차, 러너 공유 여부) 차이를 아는가?
- [ ] 캐시와 아티팩트를 언제 각각 써야 하는지 구분할 수 있는가?
- [ ] `GITHUB_TOKEN`의 권한을 최소로 설정했는가?
- [ ] 클라우드 배포에 장기 액세스 키 대신 OIDC를 사용할 수 있는가?
- [ ] `pull_request`와 `pull_request_target`의 보안 차이를 설명할 수 있는가?
- [ ] 배포 실패 시 롤백 절차가 문서화되어 있고 실제로 동작하는가?

## 참고

- [GitHub Actions — Workflow syntax](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [GitHub Actions — Events that trigger workflows](https://docs.github.com/en/actions/reference/events-that-trigger-workflows)
- [GitHub Actions — Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Actions — About security hardening with OpenID Connect](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [GitHub Actions — Caching dependencies to speed up workflows](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/cache-dependencies)
- [GitHub Actions — Using environments for deployment](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [Martin Fowler — Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
