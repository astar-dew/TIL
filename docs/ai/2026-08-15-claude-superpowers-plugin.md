---
title: "Superpowers 플러그인 뜯어보기: 스킬로 개발 방법론을 강제할 수 있을까"
date: 2026-08-15
tags: [claude-code, superpowers, skills, plugin, hooks, tdd, agent, harness-engineering]
description: "Claude Code 플러그인 Superpowers의 구조를 저장소에서 직접 확인하고, 스킬과 훅을 조합해 개발 프로세스를 유지하는 설계를 정리한다."
---

## 학습 목적

Claude Code로 작업하다 보면 "계획부터 세워줘", "테스트 먼저 써줘" 같은 지시를 매번 반복하게 된다. **개발 프로세스 자체를 도구에 심을 수 없을까**라는 발상에서 나온 것이 [Superpowers](https://github.com/obra/superpowers) 플러그인이다.

이 글은 플러그인 사용법 소개가 아니라, **그 안에서 쓸 만한 설계를 뽑아내는 것**이 목적이다. 남의 방법론을 통째로 들이지 않더라도 배울 구조가 있다.

Skills와 Rules의 기본 개념은 [Claude Code 기준으로 이해하는 Rules와 Skills](./2026-07-26-claude-code-rules-skills.md)에, 도구 활용 전반은 [Claude Code 실전 활용](./2026-08-06-claude-code-features-vibe-coding.md)에 정리했다.

## 무엇인가

한 줄로 요약하면 **"개발 방법론을 스킬 문서 묶음으로 패키징한 플러그인"** 이다.

새로운 기능을 추가하는 것이 아니다. Skills는 Claude Code의 **기본 기능**이고, Superpowers는 그 위에 **검증된 절차를 미리 짜둔 것**이다. 직접 `.claude/skills/`에 만들어도 되는 것을 잘 정리해서 배포한 형태다.

```bash
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

## 저장소에서 확인한 구조

블로그 글로만 읽으면 놓치는 부분이 있어 저장소를 직접 확인했다.

```text
superpowers/
├── skills/          14개 스킬
├── hooks/           SessionStart 훅          ← 핵심
├── docs/
├── tests/           스킬 자체를 테스트
├── .claude-plugin/  Claude Code용
├── .codex-plugin/   Codex용
├── .cursor-plugin/  Cursor용
├── .devin-plugin/   Devin용
├── AGENTS.md
└── GEMINI.md
```

여기서 두 가지가 눈에 띈다.

**하나. 이미 여러 에이전트를 지원한다.** Codex, Cursor, Devin, Gemini, Kimi, opencode용 디렉터리가 따로 있다. "Claude 전용"이라는 설명은 현재 저장소 상태와 맞지 않는다. `AGENTS.md` 같은 규격이 사실상 표준으로 자리 잡으면서 방법론을 여러 도구에 이식할 수 있게 된 결과로 보인다.

**둘. `hooks/`가 있다.** 이게 설계의 핵심인데, 뒤에서 따로 다룬다.

### 14개 스킬

| 단계 | 스킬 |
| --- | --- |
| 탐색 | `brainstorming` |
| 계획 | `writing-plans`, `executing-plans` |
| 구현 | `test-driven-development`, `using-git-worktrees` |
| 병렬화 | `dispatching-parallel-agents`, `subagent-driven-development` |
| 검증 | `requesting-code-review`, `receiving-code-review`, `verification-before-completion` |
| 마무리 | `finishing-a-development-branch` |
| 메타 | `using-superpowers`, `writing-skills` |

낱개 팁이 아니라 **브레인스토밍부터 브랜치 정리까지 이어지는 하나의 흐름**으로 짜여 있다는 점이 특징이다.

`writing-skills`가 있다는 것도 흥미롭다. **새 스킬을 만드는 방법 자체가 스킬**이라 사용자가 자기 방법론을 같은 형식으로 확장할 수 있다.

## 설계에서 배울 점 1: 훅으로 컨텍스트를 되살린다

가장 실용적인 아이디어다. `hooks/hooks.json`은 이렇게 되어 있다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "shell": "bash",
            "async": false
          }
        ]
      }
    ]
  }
}
```

`matcher`에 주목할 필요가 있다. **`startup`뿐 아니라 `clear`와 `compact`도 포함되어 있다.**

이것이 해결하는 문제는 명확하다.

```text
세션 시작 → 스킬 안내가 컨텍스트에 로드됨
  ↓
작업이 길어져 /compact 또는 /clear
  ↓
(훅이 없다면) 스킬 존재 자체를 잊음 → 원래 방식으로 회귀
  ↓
(훅이 있으면) 자동으로 다시 주입 → 유지
```

컨텍스트가 압축되거나 비워지면 초반에 주입한 지침이 사라진다. **`SessionStart` 훅에 `clear|compact` 매처를 거는 것은 이 문제를 해결하는 재사용 가능한 패턴**이다. 플러그인을 쓰지 않더라도 이 아이디어는 그대로 가져다 쓸 수 있다.

## 설계에서 배울 점 2: 지시문은 확률적이다

`test-driven-development/SKILL.md`의 실제 내용 일부다.

```markdown
**Core principle:** If you didn't watch the test fail,
you don't know if it tests the right thing.

**Violating the letter of the rules is violating the spirit of the rules.**

Thinking "skip TDD just this once"? Stop. That's rationalization.

## The Iron Law
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

Write code before the test? Delete it. Start over.
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Delete means delete
```

문체가 이례적으로 강하다. "Iron Law", "Delete means delete", "That's rationalization" 같은 표현이 이어진다.

**왜 이렇게 썼을까.** 스킬은 결국 컨텍스트에 로드되는 지시문이고, 모델은 이를 **확률적으로** 따른다. 상황이 급해 보이면 건너뛸 여지가 생긴다. 그래서 예외 상황을 미리 봉쇄하고 자기합리화의 경로를 명시적으로 차단하는 문체를 쓴 것이다.

저장소에 `tests/` 디렉터리가 있는 것도 같은 맥락이다. **스킬 문서 자체를 테스트한다.** 블로그에 소개된 압박 테스트 시나리오가 그 예다.

```text
"프로덕션 다운! 매분 $5,000 손실.
 즉시 디버깅할까, 스킬 먼저 확인할까?"
```

이런 시나리오를 서브에이전트에게 던져 스킬을 건너뛰는지 확인한다.

### 여기서 나오는 결론

**"강제(enforce)"라는 표현은 정확하지 않다.** 압박 테스트가 필요하다는 사실 자체가, 지시문만으로는 준수를 보장할 수 없다는 증거다.

수단별로 성격을 구분해야 한다.

| 수단 | 성격 | 보장 수준 |
| --- | --- | --- |
| Skill / `CLAUDE.md` | 지시문 | 확률적. 대개 따르지만 건너뛸 수 있음 |
| **Hook** | 하네스가 실행 | **결정론적** |
| **CI 검증** | 통과 못 하면 진행 불가 | **결정론적** |

Superpowers가 잘한 것은 **둘을 조합한 것**이다. 스킬로 절차를 설명하고, 훅으로 그 스킬이 잊히지 않게 한다. 스킬만으로 밀어붙이지 않았다.

진짜로 "테스트 먼저"를 보장하려면 결국 CI에 커버리지 게이트를 걸어야 한다.

## 설계에서 배울 점 3: 스킬은 조합되어야 한다

낱개 스킬 14개가 아니라 **서로를 호출하는 파이프라인**이라는 점이 이 저장소의 완성도를 보여준다.

```text
brainstorming
   ↓ 요구사항이 정리되면
writing-plans
   ↓ 계획이 서면
using-git-worktrees      (작업 격리)
   ↓
executing-plans → test-driven-development
   ↓
requesting-code-review → receiving-code-review
   ↓
verification-before-completion
   ↓
finishing-a-development-branch
```

스킬을 만들 때 하나씩 고립시켜 쓰는 것보다, **앞 단계의 산출물이 다음 단계의 입력이 되도록** 설계하면 실제로 굴러간다.

`verification-before-completion`이 별도 스킬로 있다는 점도 눈여겨볼 만하다. "다 됐습니다"라고 말하기 전에 확인하는 절차를 아예 하나의 단계로 만든 것이다.

## 이 저장소에 적용한다면

TIL 저장소 기준으로 보면 대부분은 해당이 없다.

| 스킬 | 적용 여부 |
| --- | --- |
| `test-driven-development` | **무관.** 문서 저장소라 대상이 없다 |
| `using-git-worktrees` | Claude Code에 `claude -w`로 이미 있다 |
| `writing-plans` | 플랜 모드로 대체 가능 |
| `verification-before-completion` | **유효.** `mkdocs build --strict`가 그 역할 |
| `systematic-debugging` | 다른 프로젝트에서 유효 |

그리고 이 저장소는 이미 핵심 원리를 실천하고 있다. **`mkdocs build --strict`가 링크 깨짐을 잡아주는 것**이 곧 "완료 전 검증"이고, Superpowers가 TDD로 얻으려는 것과 같은 종류의 안전장치다.

바로 가져올 만한 것은 하나다. **`SessionStart` 훅에 `clear|compact` 매처를 거는 패턴.** 프로젝트 규칙이 컨텍스트 압축 후에도 유지되게 하는 데 쓸 수 있다.

## 남의 방법론을 들일 때의 비용

Superpowers는 저자의 개발 철학이 강하게 반영된 결과물이다. TDD를 "Iron Law"로 규정하는 수준의 강도는 모든 팀에 맞지 않는다.

| 상황 | 판단 |
| --- | --- |
| 팀 코딩 표준을 세우고 싶다 | 참고할 가치 있음 |
| TDD를 배우는 중이다 | 유용. 절차가 명문화되어 있다 |
| 프로토타이핑 위주다 | 과한 프로세스가 방해가 된다 |
| 이미 자기 방식이 정립되어 있다 | 충돌 가능성 |

블로그 글에서는 "초기 스타트업 비추천"이라고 했는데, 이건 회사 단계가 아니라 **작업 크기**의 문제로 보는 편이 맞다. 스타트업이라도 결제나 인증처럼 되돌리기 어려운 기능은 계획부터 세우는 게 맞고, 대기업이라도 오타 수정에 브레인스토밍 15개 질문은 낭비다.

**권장하는 접근**은 통째로 설치하기보다 이 순서다.

1. 저장소의 `skills/*/SKILL.md`를 읽는다 (문서가 곧 방법론이다)
2. 내 작업에 맞는 것 두세 개만 골라 직접 다시 쓴다
3. 훅 설계만 참고해서 내 규칙에 적용한다

## 정리

- Superpowers는 새 기능이 아니라 **Claude Code의 기본 Skills 위에 개발 방법론을 패키징한 것**이다.
- 저장소에는 14개 스킬이 있고, 브레인스토밍부터 브랜치 정리까지 하나의 파이프라인으로 연결된다.
- 이미 Codex·Cursor·Devin·Gemini 등 여러 에이전트용 디렉터리를 갖추고 있다.
- 핵심 설계는 **`SessionStart` 훅에 `clear|compact` 매처를 걸어** 컨텍스트가 비워져도 스킬이 되살아나게 한 것이다.
- 스킬 문서를 강한 명령형으로 쓰고 압박 테스트까지 하는 이유는, **지시문이 확률적이기 때문**이다.
- 따라서 "프로세스를 강제한다"는 표현은 부정확하다. 결정론적인 것은 훅과 CI뿐이다.
- 통째로 도입하기보다 스킬 문서를 읽고 필요한 것만 내 방식으로 다시 쓰는 편이 낫다.

## 학습 체크리스트

- [ ] Skills가 Claude Code의 기본 기능이고 플러그인은 그 배포 수단임을 구분할 수 있는가?
- [ ] 지시문(Skill)과 훅(Hook)의 보장 수준 차이를 설명할 수 있는가?
- [ ] `SessionStart` 훅에 `clear|compact`를 거는 이유를 설명할 수 있는가?
- [ ] 스킬을 파이프라인으로 연결한다는 것이 무슨 의미인지 아는가?
- [ ] 담당 프로젝트에 "완료 전 검증"에 해당하는 명령이 있는가?
- [ ] 남의 방법론을 도입할 때 어디까지 가져오고 어디부터 내 것으로 바꿀지 기준이 있는가?

## 참고

- [obra/superpowers](https://github.com/obra/superpowers) — 스킬 원문
- [obra/superpowers-marketplace](https://github.com/obra/superpowers-marketplace) — 플러그인 마켓플레이스
- [Claude Code — Skills](https://code.claude.com/docs/en/skills)
- [Claude Code — Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code — Plugins](https://code.claude.com/docs/en/plugins)
- [곰국 블로그 — Claude Superpowers 플러그인 완벽 정리](https://gomguk.tistory.com/313)
