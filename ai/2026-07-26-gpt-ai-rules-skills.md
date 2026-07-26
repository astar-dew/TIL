---
layout: post
title: "GPT 기준으로 이해하는 AI Rules와 Skills"
date: 2026-07-26
categories: [ai]
tags: [gpt, codex, rules, skills, agents, harness-engineering]
description: "GPT와 Codex를 기준으로 Rule, Skill, Prompt의 차이와 프로젝트 적용 방법을 정리한다."
---

## 학습 목적

AI에게 같은 프로젝트 설명을 반복하면 요청이 길어지고 결과도 일정하지 않다.
프로젝트 규칙과 반복 작업 절차를 미리 구성하면 짧은 요청으로도 일관된 결과를 얻을 수 있다.

이처럼 AI가 안정적으로 일하도록 규칙, 도구, 맥락, 검증 절차를 제공하는 환경 설계를
**하네스 엔지니어링(Harness Engineering)**이라고 볼 수 있다.

## 먼저 알아둘 점

GPT 모델 자체가 특정 `rules` 폴더나 파일을 공통 표준으로 읽는 것은 아니다.
규칙과 스킬을 어떤 형식으로 제공하는지는 GPT를 사용하는 제품이나 에이전트에 따라 다르다.

- ChatGPT: 현재 대화의 요청, 프로젝트 지침, 사용자 지정 지침 등을 사용한다.
- Codex: 저장소의 `AGENTS.md`를 지속적인 프로젝트 지침으로 사용한다.
- ChatGPT와 Codex의 Skill: `SKILL.md`를 중심으로 지침, 참고 문서, 스크립트를 묶어 재사용한다.

이 글에서는 구체적인 파일 구조를 사용할 수 있는 **Codex 기준**으로 설명한다.

## 핵심 개념

### Prompt

현재 작업에서만 필요한 일회성 요청이다.

```text
Expo Metro 오류 해결 과정을 트러블슈팅 글로 작성해줘.
```

작업 대상과 이번에 원하는 결과처럼 매번 달라지는 내용을 적는다.

### Rule

프로젝트에서 계속 지켜야 하는 기준이다.
Codex에서는 저장소의 `AGENTS.md`가 이 역할을 담당한다.

```md
# AGENTS.md

- 답변과 문서는 한국어로 작성한다.
- 학습 글은 Jekyll front matter를 포함한다.
- 트러블슈팅 글은 현상, 원인, 해결, 검증 순서로 작성한다.
- 변경한 Markdown 파일은 링크와 형식을 확인한다.
```

Rule에는 다음과 같은 내용을 넣는 것이 적합하다.

- 저장소 구조와 파일 위치
- 코딩 및 문서 작성 규칙
- 빌드, 테스트, 검증 명령어
- 반복해서 발생한 실수의 방지 기준
- 반드시 지켜야 하는 리뷰 기준

### Skill

특정 작업을 수행하는 재사용 가능한 워크플로다.
Codex에서는 `.agents/skills/<skill-name>/SKILL.md` 형태로 저장소에 둘 수 있다.

```text
.agents/
└── skills/
    └── write-troubleshooting-post/
        ├── SKILL.md
        ├── references/
        │   └── writing-guide.md
        └── scripts/
            └── validate-front-matter.sh
```

`SKILL.md`에는 언제 이 Skill을 사용해야 하는지와 작업 순서를 작성한다.

```md
---
name: write-troubleshooting-post
description: 개발 오류의 현상, 원인, 해결, 검증 과정을 TIL 글로 작성할 때 사용한다.
---

1. 사용자가 제공한 오류 메시지와 실행 환경을 확인한다.
2. `_templates/troubleshooting.md` 형식을 사용한다.
3. 확인된 사실과 추정 내용을 구분한다.
4. 재현 및 해결 명령어를 코드 블록으로 작성한다.
5. 카테고리 README에 글 링크를 추가한다.
6. Markdown 형식과 내부 링크를 검증한다.
```

필요하면 Skill에 스크립트와 참고 자료를 포함할 수 있다.
단순한 설명 모음보다, 입력부터 검증까지 일정한 순서가 있는 작업에 적합하다.

## Prompt, Rule, Skill 비교

| 구분 | 목적 | 적용 범위 | 저장 위치 예시 | 예시 |
| --- | --- | --- | --- | --- |
| Prompt | 현재 작업 요청 | 현재 대화 또는 작업 | 사용자 메시지 | "Metro 오류 글을 작성해줘" |
| Rule | 항상 지킬 기준 | 저장소 또는 하위 디렉터리 | `AGENTS.md` | "글은 한국어로 작성한다" |
| Skill | 반복 작업 절차 | 특정 작업이 선택된 경우 | `.agents/skills/*/SKILL.md` | "트러블슈팅 글 작성 절차" |

간단히 구분하면 다음과 같다.

```text
Prompt = 이번에 무엇을 할 것인가
Rule   = 작업할 때 무엇을 항상 지킬 것인가
Skill  = 그 작업을 어떤 순서와 도구로 수행할 것인가
```

## GPT/Codex가 작업하는 흐름

1. 사용자가 현재 작업을 Prompt로 요청한다.
2. Codex가 작업 경로에 적용되는 `AGENTS.md`를 읽는다.
3. 요청과 설명이 일치하는 Skill을 선택한다.
4. 선택한 Skill의 전체 지침과 필요한 참고 자료를 읽는다.
5. 파일을 수정하고 Rule과 Skill에 정의된 검증을 수행한다.

Skill은 처음부터 모든 내용을 대화에 넣지 않고 이름과 설명을 먼저 노출한 뒤,
필요할 때 전체 지침을 읽는 방식으로 컨텍스트 사용량을 줄일 수 있다.

## 언제 무엇을 사용해야 하는가

### Rule이 적합한 경우

- AI가 같은 프로젝트 규칙을 반복해서 어긴다.
- 팀원이 요청하는 방식과 관계없이 결과 형식이 같아야 한다.
- 특정 폴더에서만 다른 규칙을 적용해야 한다.
- 매 작업마다 같은 테스트나 검증을 수행해야 한다.

### Skill이 적합한 경우

- 여러 단계가 항상 같은 순서로 반복된다.
- 여러 파일을 정해진 형식으로 생성해야 한다.
- 특정 참고 문서나 템플릿을 매번 사용한다.
- 스크립트나 외부 도구 실행이 작업에 포함된다.

### Prompt만으로 충분한 경우

- 한 번만 수행할 작업이다.
- 요구사항이 아직 자주 바뀐다.
- 반복 가능한 절차가 정리되지 않았다.

처음부터 모든 작업을 Rule이나 Skill로 만들 필요는 없다.
일회성 Prompt로 시작하고, 반복되는 기준은 Rule로, 안정된 절차는 Skill로 옮기는 방식이 효율적이다.

## 이 TIL 프로젝트에 적용하는 예시

이 저장소에서는 다음과 같이 역할을 나눌 수 있다.

```text
AGENTS.md
└── 한국어 작성, 간결한 답변, 공통 문서 규칙

_templates/
├── learning-note.md
├── troubleshooting.md
└── development-log.md

.agents/skills/
├── write-learning-note/
├── write-troubleshooting-post/
└── write-development-log/
```

- `AGENTS.md`: 모든 글에 적용할 공통 기준
- `_templates`: 글에 들어갈 실제 Markdown 구조
- `SKILL.md`: 템플릿 선택, 자료 조사, 파일 생성, 링크 추가, 검증 순서

예를 들어 사용자는 다음처럼 짧게 요청할 수 있다.

```text
트러블슈팅 Skill로 Expo Metro 번들 오류를 정리해줘.
```

Skill이 잘 구성되어 있다면 AI는 템플릿 선택부터 카테고리 링크 추가와 검증까지
정해진 흐름으로 수행할 수 있다.

## 작성 및 운영 원칙

### 이미 알려진 일반 지식은 Rule에 반복하지 않는다

Rule에는 TypeScript 문법처럼 모델이 이미 알고 있는 내용보다
프로젝트 고유의 구조, 선택, 예외를 적는다.

### Rule과 Skill의 내용을 중복하지 않는다

공통 기준은 Rule에 두고, 특정 작업의 단계는 Skill에 둔다.
중복된 지침은 수정할 곳을 늘리고 서로 충돌할 가능성을 만든다.

### 설명은 짧고 구체적으로 작성한다

`좋은 코드를 작성한다`보다 `API 요청은 UserAPI 클래스를 사용한다`처럼
결과를 확인할 수 있는 기준이 효과적이다.

### 자동 검증을 함께 둔다

AI 지침만으로 규칙 준수를 완전히 보장할 수는 없다.
린터, 타입 검사, 테스트, 링크 검사처럼 기계적으로 확인할 수 있는 항목은 자동화한다.

### 실제 반복 문제를 기준으로 개선한다

예상만으로 많은 규칙을 만들면 컨텍스트와 유지보수 비용이 증가한다.
반복되는 수정 요청이나 리뷰 의견이 생겼을 때 Rule 또는 Skill에 반영한다.

## 정리

- Rule은 AI가 매번 지켜야 하는 프로젝트 기준이다.
- Skill은 특정 작업을 안정적으로 반복하기 위한 절차와 자료의 묶음이다.
- Prompt는 이번 작업의 목표와 입력값을 전달한다.
- Codex에서는 Rule을 `AGENTS.md`, Skill을 `.agents/skills/*/SKILL.md`로 관리할 수 있다.
- 좋은 AI 환경은 긴 프롬프트보다 정확한 규칙, 재사용 가능한 절차, 자동 검증으로 만들어진다.

## 참고

- [우아한형제들 기술블로그 - 하네스 엔지니어링으로 팀 맞춤형 AI 환경 구축하기](https://techblog.woowahan.com/26177/)
- [OpenAI Codex - Customization](https://developers.openai.com/codex/concepts/customization)
- [OpenAI Codex - Build skills](https://developers.openai.com/codex/skills)
- [Agent Skills Specification](https://agentskills.io/specification)
