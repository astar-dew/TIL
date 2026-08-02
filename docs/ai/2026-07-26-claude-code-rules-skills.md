---
title: "Claude Code 기준으로 이해하는 Rules와 Skills"
date: 2026-07-26
tags: [claude, claude-code, rules, skills, agents, harness-engineering]
description: "Claude Code의 CLAUDE.md, 경로별 Rules, Skills, settings 설정과 역할을 정리한다."
---

## 학습 목적

Claude Code가 프로젝트 규칙을 일관되게 따르고 반복 작업을 정해진 순서로 수행하도록
지침과 Skill을 구성하는 방법을 정리한다.

이 글은 Claude API를 직접 호출하는 일반적인 애플리케이션이 아니라,
로컬 저장소에서 파일과 명령어를 다루는 **Claude Code 기준**이다.

## 전체 구조

Claude Code에서는 다음과 같이 역할을 나눌 수 있다.

```text
project/
├── CLAUDE.md
├── CLAUDE.local.md
└── .claude/
    ├── settings.json
    ├── settings.local.json
    ├── rules/
    │   ├── markdown.md
    │   └── frontend.md
    └── skills/
        └── write-til-post/
            ├── SKILL.md
            ├── references/
            └── scripts/
```

| 구성 | 역할 | 로드 시점 |
| --- | --- | --- |
| `CLAUDE.md` | 프로젝트 전체에서 항상 필요한 핵심 지침 | 대화 시작 시 |
| `.claude/rules/*.md` | 주제 또는 파일 경로별 세부 규칙 | 항상 또는 대상 파일 작업 시 |
| `.claude/skills/*/SKILL.md` | 참고 자료와 반복 작업 워크플로 | Skill이 선택되거나 호출될 때 |
| `.claude/settings.json` | 권한, 환경, 도구 동작 설정 | Claude Code 실행 시 |
| `CLAUDE.local.md` | 공유하지 않을 개인 프로젝트 지침 | 대화 시작 시 |

## CLAUDE.md

`CLAUDE.md`는 Claude Code가 프로젝트를 이해하기 위해 매 세션 읽는 지침 파일이다.
새로운 팀원에게 설명할 프로젝트 핵심 정보라고 생각하면 된다.

```md
# Project Guide

- 문서와 답변은 한국어로 작성한다.
- 학습 글은 front matter에 `title`, `date`, `tags`, `description`을 포함한다.
- 글 파일은 `docs/` 아래 주제에 맞는 카테고리 디렉터리에 생성한다.
- 새 글을 만들면 해당 카테고리의 `index.md`에 링크를 추가한다.
- Markdown 변경 후 `git diff --check`를 실행한다.
```

다음 내용이 적합하다.

- 빌드, 테스트, 검증 명령어
- 프로젝트 구조와 주요 파일 위치
- 코딩 및 문서 작성 규칙
- 팀에서 사용하는 아키텍처와 도구
- 모든 작업에서 반복되는 주의사항

`CLAUDE.md`가 너무 길면 개별 지침이 잘 적용되지 않을 수 있다.
항상 필요한 핵심만 유지하고, 경로별 규칙은 Rules로, 작업 절차는 Skills로 분리한다.

### 적용 범위

| 범위 | 위치 | 용도 |
| --- | --- | --- |
| 사용자 전체 | `~/.claude/CLAUDE.md` | 모든 프로젝트에서 사용할 개인 기준 |
| 프로젝트 | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 팀이 공유할 프로젝트 기준 |
| 개인 프로젝트 | `./CLAUDE.local.md` | 현재 프로젝트에서만 사용할 개인 설정 |

`CLAUDE.local.md`에는 개인 테스트 주소나 로컬 환경 정보처럼
팀과 공유할 필요가 없는 내용을 작성하고 `.gitignore`에 추가한다.

## Rules

프로젝트가 커지면 모든 규칙을 `CLAUDE.md` 하나에 넣기 어렵다.
이때 `.claude/rules/` 아래에 주제별 Markdown 파일을 만든다.

```text
.claude/rules/
├── markdown.md
├── testing.md
├── frontend/
│   └── react.md
└── backend/
    └── api.md
```

`paths`가 없는 Rule은 모든 세션에 적용된다.

```md
# Markdown Rules

- 제목은 글의 문제나 학습 주제를 구체적으로 표현한다.
- 명령어는 언어가 지정된 코드 블록으로 작성한다.
- 확인하지 않은 내용은 사실처럼 단정하지 않는다.
```

### 경로별 Rule

YAML front matter의 `paths`에 glob 패턴을 지정하면
Claude가 일치하는 파일을 작업할 때만 해당 Rule이 적용된다.

```md
---
paths:
  - "docs/app/**/*.md"
  - "docs/web/**/*.md"
---

# Development Post Rules

- front matter에 `title`, `date`, `tags`, `description`을 작성한다.
- 코드 예제에는 실행 환경이나 파일 위치를 함께 적는다.
- 카테고리 `index.md`에 새 글의 링크를 추가한다.
```

경로별 Rule은 관련 없는 작업에 세부 지침이 포함되는 것을 줄여
컨텍스트를 더 효율적으로 사용할 수 있게 한다.

## Skills

Skill은 Claude가 필요할 때 불러오는 참고 자료 또는 반복 작업 절차다.
프로젝트 Skill은 `.claude/skills/<skill-name>/SKILL.md`에 작성한다.

```text
.claude/skills/write-til-post/
├── SKILL.md
├── references/
│   └── writing-guide.md
└── scripts/
    └── validate-post.sh
```

Skill은 두 가지 방식으로 실행할 수 있다.

- 자동 실행: 사용자 요청과 `description`이 일치하면 Claude가 선택한다.
- 직접 실행: `/write-til-post`처럼 Skill 이름을 명령으로 호출한다.

### 기본 SKILL.md 예시

```md
---
name: write-til-post
description: 개발 학습 내용이나 오류 해결 과정을 TIL의 Markdown 글로 작성할 때 사용한다.
argument-hint: "[주제] [템플릿]"
---

# TIL 글 작성

1. 사용자가 제공한 사실, 오류 메시지, 실행 환경을 확인한다.
2. 목적에 따라 `_templates/`에서 학습, 트러블슈팅, 개발 일지 템플릿을 선택한다.
3. `docs/` 아래 주제에 맞는 카테고리 디렉터리에 날짜와 제목으로 파일을 생성한다.
4. 카테고리 `index.md`에 글 링크를 추가한다.
5. `mkdocs build --strict`로 front matter와 내부 링크를 검증한다.

상세 작성 기준이 필요하면 [writing-guide.md](references/writing-guide.md)를 읽는다.
```

호출 예시는 다음과 같다.

```text
/write-til-post "Claude Code Skills" "학습 정리"
```

### Skill 저장 위치

| 범위 | 위치 | 적용 대상 |
| --- | --- | --- |
| 개인 | `~/.claude/skills/<name>/SKILL.md` | 사용자의 모든 프로젝트 |
| 프로젝트 | `.claude/skills/<name>/SKILL.md` | 현재 저장소 |
| 플러그인 | `<plugin>/skills/<name>/SKILL.md` | 플러그인이 활성화된 환경 |

기존 `.claude/commands/*.md`도 계속 사용할 수 있지만,
지원 파일과 자동 선택 등 더 많은 기능을 제공하는 Skills 사용이 권장된다.

## 주요 Skill 설정

`SKILL.md`의 YAML front matter로 호출과 실행 방식을 설정할 수 있다.

| 설정 | 설명 |
| --- | --- |
| `name` | Skill 이름이며 생략하면 디렉터리 이름을 사용한다. |
| `description` | 무엇을 하고 언제 사용하는지 설명하며 자동 선택 기준이 된다. |
| `argument-hint` | 직접 호출할 때 필요한 인자를 안내한다. |
| `disable-model-invocation` | `true`이면 Claude의 자동 호출을 막고 사용자만 실행한다. |
| `user-invocable` | `false`이면 `/` 메뉴에서 숨긴다. |
| `allowed-tools` | Skill 실행 중 승인 없이 사용할 수 있는 도구를 지정한다. |
| `paths` | 특정 파일 경로에서만 자동으로 선택되도록 제한한다. |
| `context` | `fork`이면 별도의 하위 에이전트 컨텍스트에서 실행한다. |
| `agent` | `context: fork`일 때 사용할 에이전트를 지정한다. |

배포처럼 사용자가 명시적으로 시작해야 하는 작업은 자동 호출을 막는 것이 안전하다.

```md
---
name: deploy
description: 테스트와 빌드를 수행한 후 운영 환경에 배포한다.
disable-model-invocation: true
argument-hint: "[environment]"
---

1. 배포 환경을 확인한다.
2. 테스트와 빌드를 실행한다.
3. 변경 내용과 배포 대상을 사용자에게 보여준다.
4. 승인된 경우에만 배포한다.
```

`allowed-tools`는 지정하지 않은 도구를 차단하는 설정이 아니다.
나열한 도구를 추가 승인 없이 사용할 수 있게 하는 설정이므로,
실제로 금지할 작업은 `.claude/settings.json`의 `permissions.deny`로 관리해야 한다.

## settings.json

Claude Code의 권한과 도구 동작은 JSON 설정으로 관리한다.

| 위치 | 용도 |
| --- | --- |
| `~/.claude/settings.json` | 사용자 전체 설정 |
| `.claude/settings.json` | 저장소에서 공유할 프로젝트 설정 |
| `.claude/settings.local.json` | 공유하지 않을 개인 프로젝트 설정 |

민감한 파일 접근 차단 예시는 다음과 같다.

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)"
    ]
  }
}
```

Skill 표시 상태는 `skillOverrides`로 조정할 수 있다.

```json
{
  "skillOverrides": {
    "write-til-post": "on",
    "legacy-post-writer": "off",
    "deploy": "user-invocable-only"
  }
}
```

- `on`: Claude와 `/` 메뉴에 표시한다.
- `name-only`: Claude에는 이름만 제공하고 `/` 메뉴에는 표시한다.
- `user-invocable-only`: Claude에서 숨기고 `/` 메뉴에서만 실행할 수 있다.
- `off`: Claude와 `/` 메뉴 모두에서 숨긴다.

## 기존 AGENTS.md와 함께 사용하기

Claude Code는 `AGENTS.md`를 기본 지침 파일로 직접 읽지 않고 `CLAUDE.md`를 사용한다.
Codex와 Claude Code가 같은 저장소 규칙을 공유한다면 내용을 복사하지 말고 가져올 수 있다.

```md
# CLAUDE.md

@AGENTS.md

## Claude Code

- Claude Code 전용 Skill은 `.claude/skills/`에서 관리한다.
- `/memory`와 `/skills`로 설정이 로드되었는지 확인한다.
```

공통 규칙은 `AGENTS.md`에 두고 Claude 전용 내용만 `CLAUDE.md`에 추가하면
두 파일의 내용이 서로 달라지는 문제를 줄일 수 있다.

## Codex와 Claude Code 비교

| 목적 | Codex | Claude Code |
| --- | --- | --- |
| 프로젝트 공통 지침 | `AGENTS.md` | `CLAUDE.md` |
| 경로별 지침 | 하위 디렉터리의 `AGENTS.md` | `.claude/rules/*.md`의 `paths` |
| 프로젝트 Skill | `.agents/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| 직접 Skill 호출 | `$skill-name` 또는 Skill 선택 | `/skill-name` |
| 개인 Skill | `~/.agents/skills/` | `~/.claude/skills/` |

두 도구 모두 Agent Skills의 `SKILL.md` 구조를 사용하지만,
저장 위치와 호출 방식, 추가 front matter 기능은 서로 다를 수 있다.

## 설정 확인과 문제 해결

Claude Code에서 다음 명령으로 실제 로드 상태를 확인할 수 있다.

```text
/memory       # CLAUDE.md와 Rules 확인
/skills       # 사용 가능한 Skills 확인
/permissions  # 적용된 권한 확인
/doctor       # 잘못된 설정과 설치 상태 진단
/context      # 현재 컨텍스트 구성 확인
```

Skill이 자동으로 선택되지 않으면 먼저 `description`에
작업 목적과 호출되어야 할 상황이 구체적으로 적혀 있는지 확인한다.

## 정리

- `CLAUDE.md`에는 모든 세션에서 필요한 핵심 프로젝트 지침을 작성한다.
- `.claude/rules/`에는 주제별 또는 경로별 세부 규칙을 작성한다.
- `.claude/skills/`에는 필요할 때만 사용하는 참고 자료와 반복 작업 절차를 작성한다.
- 보안상 반드시 차단할 작업은 지침이 아니라 `permissions.deny`로 설정한다.
- Codex와 함께 사용할 때는 `CLAUDE.md`에서 `@AGENTS.md`를 가져와 공통 규칙을 공유할 수 있다.

## 참고

- [Claude Code - Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Claude Code - How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Claude Code - Extend Claude Code](https://code.claude.com/docs/en/features-overview)
- [Claude Code - Settings](https://code.claude.com/docs/en/settings)
- [Claude Code - Debug your configuration](https://code.claude.com/docs/en/debug-your-config)
