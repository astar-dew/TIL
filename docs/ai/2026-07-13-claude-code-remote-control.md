---
title: "Claude Code Remote Control 연동과 활용 방법"
date: 2026-07-13
tags: [claude, claude-code, remote-control, mobile, remote-development]
description: "로컬 Claude Code 세션을 웹과 모바일에서 이어서 사용하는 Remote Control의 연동 방법, 활용 사례, 제한 사항을 정리한다."
---

## Remote Control이란

Claude Code Remote Control은 컴퓨터에서 실행 중인 Claude Code 세션을
휴대폰, 태블릿 또는 다른 컴퓨터의 브라우저에서 이어서 사용하는 기능이다.

중요한 점은 Claude Code가 클라우드로 이동하는 것이 아니라는 것이다.

```text
휴대폰 또는 브라우저
        ↓
claude.ai를 통한 연결
        ↓
내 컴퓨터에서 실행 중인 Claude Code
        ↓
로컬 파일, 명령어, MCP, 프로젝트 설정
```

- 파일 읽기와 명령어 실행은 기존 컴퓨터에서 처리된다.
- 로컬 파일 시스템, MCP 서버, 도구, 프로젝트 설정을 그대로 사용한다.
- 웹과 모바일은 로컬 세션을 확인하고 제어하는 화면 역할을 한다.
- 터미널, 브라우저, 모바일에서 같은 대화를 이어갈 수 있다.

현재 Remote Control은 연구 미리보기 단계이므로 기능과 설정이 변경될 수 있다.

## 이런 사람에게 유용하다

### 오래 걸리는 작업을 자주 실행하는 개발자

테스트, 빌드, 코드 분석, 여러 파일 리팩토링을 실행한 뒤
컴퓨터 앞을 떠나도 진행 상황과 결과를 확인할 수 있다.

```text
대규모 테스트를 실행해줘.
완료되거나 내 결정이 필요하면 알려줘.
```

### 로컬 개발 환경을 그대로 사용해야 하는 개발자

로컬 데이터베이스, 사내 MCP 서버, 개발용 인증, 설치된 도구처럼
클라우드 환경으로 옮기기 어려운 구성을 유지하면서 다른 기기에서 작업할 수 있다.

### 이동 중에 확인과 승인만 처리하려는 사용자

휴대폰에서 Claude의 질문에 답하거나 다음 작업을 지시할 수 있다.
코드를 직접 작성하기보다 진행 상황 확인과 의사결정에 특히 적합하다.

### 여러 장소에서 같은 작업을 이어가는 사용자

회사 컴퓨터에서 시작한 작업을 다른 컴퓨터의 브라우저나 휴대폰에서 이어갈 수 있다.
대화와 작업 상태가 연결된 기기 사이에서 동기화된다.

## 사용 전 요구 사항

- Claude Pro, Max, Team 또는 Enterprise 구독이 필요하다.
- API Key 인증만으로는 사용할 수 없다.
- Team과 Enterprise는 관리자가 Remote Control을 활성화해야 한다.
- Claude Code에서 claude.ai 계정으로 로그인해야 한다.
- 프로젝트 디렉터리의 작업 공간 신뢰를 한 번 이상 승인해야 한다.
- Claude Code가 `api.anthropic.com`을 사용해야 한다.
- Remote Control을 실행하는 컴퓨터가 켜져 있고 Claude Code 프로세스가 실행 중이어야 한다.

Amazon Bedrock, Google Cloud의 Agent Platform, Microsoft Foundry에서는 사용할 수 없다.
`ANTHROPIC_BASE_URL`이 프록시나 별도 LLM Gateway를 가리키는 환경에서도 비활성화된다.

## 가장 간단한 연동 방법

### 1. 프로젝트에서 Claude Code 로그인 확인

```bash
cd path/to/project
claude
```

로그인이 필요하면 Claude Code에서 다음 명령을 실행한다.

```text
/login
```

처음 실행하는 프로젝트라면 작업 공간 신뢰 안내를 확인하고 승인한다.

### 2. Remote Control 시작

작업 방식에 따라 다음 중 하나를 선택한다.

#### 원격 연결을 기다리는 서버 모드

```bash
claude remote-control --name "TIL"
```

터미널은 서버처럼 실행되며 세션 URL을 보여준다.
스페이스바를 누르면 휴대폰에서 스캔할 수 있는 QR 코드를 표시할 수 있다.

#### 터미널과 원격 기기를 함께 사용하는 대화형 모드

```bash
claude --remote-control "TIL"
```

터미널에서 Claude Code를 평소처럼 사용하면서
같은 세션을 웹이나 모바일에서도 제어할 수 있다.

#### 이미 진행 중인 세션에서 연결

```text
/remote-control TIL
```

짧은 명령인 `/rc TIL`도 사용할 수 있다.
현재 대화 기록을 유지한 상태로 Remote Control을 활성화한다.

VS Code 확장에서도 프롬프트 입력란에 `/remote-control` 또는 `/rc`를 입력할 수 있다.

### 3. 다른 기기에서 접속

다음 방법 중 하나로 연결한다.

- 터미널에 표시된 세션 URL을 브라우저에서 연다.
- 터미널의 QR 코드를 Claude 모바일 앱으로 스캔한다.
- [claude.ai/code](https://claude.ai/code)의 세션 목록에서 이름을 선택한다.
- Claude 모바일 앱의 코드 메뉴에서 활성 세션을 선택한다.

Remote Control 세션은 세션 목록에서 컴퓨터 아이콘과 온라인 상태로 표시된다.

## 주요 활용 방법

### 작업 진행 상황 확인

터미널을 직접 열지 않고도 현재 대화, 도구 실행, Subagent와 Workflow의 진행 상태를 확인한다.

### 모바일에서 추가 작업 요청

```text
실패한 테스트의 원인을 분석하고 수정 방향만 정리해줘.
```

```text
변경 파일을 다시 검토하고 보안상 위험한 부분이 있으면 멈춰줘.
```

터미널과 모바일에서 번갈아 메시지를 보내도 같은 대화에 반영된다.

### 파일과 이미지 전달

Claude 앱이나 claude.ai/code에서 이미지 또는 파일을 첨부하면
로컬 Claude Code가 컴퓨터로 내려받아 작업에 사용할 수 있다.

예를 들어 이동 중 발견한 오류 화면을 첨부하고
로컬 프로젝트의 관련 코드를 분석하도록 요청할 수 있다.

### 푸시 알림 사용

Claude 모바일 앱에서 알림을 허용하고 Claude Code의 `/config`에서
Remote Control 푸시 알림을 활성화할 수 있다.

```text
전체 테스트가 끝나거나 승인이 필요하면 휴대폰으로 알려줘.
```

오래 걸리는 작업 완료, 권한 요청, 추가 판단이 필요한 상황을 확인할 때 유용하다.

## 여러 작업을 동시에 실행할 때

서버 모드는 필요할 때 여러 세션을 생성할 수 있다.
기본 `same-dir` 모드는 여러 세션이 같은 디렉터리를 사용하므로
동일한 파일을 수정하면 충돌할 수 있다.

Git 저장소에서 병렬 작업을 실행한다면 worktree 모드를 고려한다.

```bash
claude remote-control --spawn worktree --capacity 3
```

- `same-dir`: 모든 세션이 현재 디렉터리를 공유한다.
- `worktree`: 각 세션이 별도의 Git worktree를 사용한다.
- `session`: 하나의 세션만 제공하고 추가 연결을 거부한다.
- `--capacity`: 동시에 실행할 수 있는 최대 세션 수를 지정한다.

병렬 작업이 필요하지 않다면 일반 대화형 모드가 가장 단순하다.

## Remote Control과 웹의 Claude Code 비교

| 구분 | Remote Control | 웹의 Claude Code |
| --- | --- | --- |
| 실행 위치 | 사용자의 컴퓨터 | Anthropic 관리 클라우드 |
| 로컬 파일 | 현재 프로젝트를 직접 사용 | 클라우드 환경에 저장소 준비 필요 |
| 로컬 MCP와 도구 | 그대로 사용 가능 | 클라우드에 별도 설정 필요 |
| 컴퓨터 상태 | 켜져 있어야 함 | 로컬 컴퓨터와 무관 |
| 적합한 상황 | 진행 중인 로컬 작업을 다른 기기에서 계속할 때 | 로컬 설정 없이 새 작업을 시작할 때 |

이미 로컬 환경에서 작업 중이고 그 환경을 유지해야 한다면 Remote Control이 적합하다.
컴퓨터를 켜둘 수 없거나 클라우드에서 독립적으로 실행하려면 웹의 Claude Code가 적합하다.

## 연결과 보안

Remote Control을 사용한다고 컴퓨터에 외부 접속용 포트가 열리지는 않는다.
로컬 Claude Code는 Anthropic API로 아웃바운드 HTTPS 연결을 만들고 작업을 확인한다.

- 모든 통신은 TLS를 통해 Anthropic API를 거친다.
- 코드 실행과 파일 시스템 접근은 로컬 컴퓨터에서 이루어진다.
- 세션 연결에는 용도가 제한된 단기 자격 증명이 사용된다.
- 메시지, 응답, 도구 활동을 포함한 세션 기록은 기기 동기화를 위해 Anthropic 서버에 저장된다.
- Zero Data Retention이 필요한 조직에서는 Remote Control을 활성화할 수 없다.

로컬에서 실행된다는 것이 자동으로 안전하다는 의미는 아니다.
원격에서 승인한 명령도 실제 컴퓨터의 파일과 프로세스에 영향을 준다.

다음 기준을 함께 적용하는 것이 좋다.

- 삭제, 배포, 데이터 변경 작업은 실행 전에 내용을 확인한다.
- 민감한 저장소에서는 Claude Code 권한과 `permissions.deny`를 설정한다.
- 필요한 경우 `claude remote-control --sandbox`로 격리를 활성화한다.
- 공용 기기에서 세션 URL과 로그인 상태를 남기지 않는다.
- 분실한 기기는 Claude 계정 설정의 신뢰할 수 있는 기기 목록에서 제거한다.

## 제한 사항

- 일반 대화형 Claude Code 프로세스 하나에는 원격 세션 하나만 연결할 수 있다.
- 터미널, VS Code 또는 Claude Code 프로세스를 종료하면 원격 세션도 종료된다.
- 컴퓨터가 장시간 오프라인이면 세션이 종료될 수 있다.
- 일부 명령은 로컬 터미널에서만 동작한다.
- 연구 미리보기 기능이므로 조직 정책이나 버전에 따라 사용할 수 없을 수 있다.

Remote Control은 컴퓨터를 원격 개발 서버로 바꾸는 기능이 아니다.
로컬에서 실행 중인 Claude Code 세션을 다른 화면에서 이어서 사용하는 기능이다.

## 연결이 되지 않을 때

### 구독 오류

API Key가 아닌 Claude 구독 계정으로 로그인했는지 확인한다.

```text
/login
```

### Team 또는 Enterprise에서 비활성화

조직 관리자가 Claude Code 관리자 설정에서 Remote Control을 활성화해야 한다.

### 지원하지 않는 API Endpoint

`ANTHROPIC_BASE_URL`이 별도 프록시를 가리키고 있는지 확인한다.
Remote Control을 사용할 때는 `api.anthropic.com`을 사용해야 한다.

### 프로젝트가 표시되지 않음

프로젝트 디렉터리에서 `claude`를 직접 실행하고 작업 공간 신뢰를 승인한다.
그다음 Remote Control을 다시 시작한다.

### 연결이 끊김

로컬 컴퓨터의 전원, 네트워크, Claude Code 프로세스가 실행 중인지 확인한다.
필요하면 다음 명령으로 새 세션을 시작한다.

```bash
claude remote-control --name "TIL"
```

## 빠른 체크리스트

- [ ] Claude 구독 계정으로 로그인했는가?
- [ ] Team 또는 Enterprise 관리자가 기능을 허용했는가?
- [ ] 프로젝트의 작업 공간 신뢰를 승인했는가?
- [ ] 로컬 Claude Code 프로세스가 계속 실행 중인가?
- [ ] 세션 URL 또는 QR 코드로 같은 계정에 접속했는가?
- [ ] 민감한 명령에 대한 권한과 승인 기준을 설정했는가?
- [ ] 세션 기록의 서버 저장이 조직 정책에 맞는가?

## 정리

Claude Code Remote Control은 로컬 개발 환경을 그대로 유지하면서
휴대폰이나 브라우저에서 작업을 이어갈 수 있게 한다.

오래 걸리는 작업 확인, 이동 중 승인, 오류 화면 전달에는 유용하지만
로컬 컴퓨터와 Claude Code 프로세스가 계속 실행되어야 한다.
또한 세션 기록이 Anthropic 서버에 저장되므로 조직의 보안 및 데이터 보존 정책을 먼저 확인해야 한다.

## 참고

- [Claude Code 공식 문서 - Remote Control](https://code.claude.com/docs/ko/remote-control)

공식 문서 확인일: 2026-07-26
