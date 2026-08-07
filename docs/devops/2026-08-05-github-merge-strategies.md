---
title: "GitHub 머지 전략 4가지: Merge Commit, Squash, Rebase, Fast-forward"
date: 2026-08-05
tags: [git, github, merge, rebase, squash, fast-forward, pull-request, branch-strategy]
description: "같은 상황에 네 가지 머지 방식을 적용해 히스토리가 어떻게 달라지는지 비교하고, 되돌리기 난이도와 팀 상황에 따른 선택 기준을 정리한다."
---

## 학습 목적

PR을 머지할 때 GitHub은 버튼 옆에 선택지를 준다. 아무거나 눌러도 코드는 합쳐지지만, **무엇을 고르느냐에 따라 히스토리 모양과 나중에 되돌리는 난이도가 완전히 달라진다.**

이 글에서는 하나의 동일한 상황을 놓고 네 가지 방식을 각각 적용해, 결과가 어떻게 달라지는지 그림으로 비교한다. 그리고 어떤 상황에 무엇을 고를지 기준을 정리한다.

커밋 히스토리가 DAG이고 커밋 해시가 부모 해시를 포함해 계산된다는 전제를 알고 있으면 이해가 빠르다. 이 부분은 [Git의 내부 구조](./2026-08-04-git-object-model-merkle-tree.md)에 정리했다.

## 먼저 짚을 것: GitHub 버튼에는 3개뿐이다

네 가지를 나란히 놓고 보기 전에 구분해야 할 것이 있다.

| 방식 | GitHub PR 버튼 | 비고 |
| --- | --- | --- |
| Merge Commit (3-way) | **있음** (Create a merge commit) | |
| Squash and Merge | **있음** | |
| Rebase and Merge | **있음** | |
| Fast-forward Merge | **없음** | 로컬 `git merge --ff-only`, GitLab에는 옵션으로 존재 |

**GitHub PR 머지는 fast-forward를 하지 않는다.** "Rebase and merge"가 결과적으로 선형 히스토리를 만들어서 fast-forward처럼 보이지만, 커밋을 복사해 **새 해시를 만든다**는 점에서 다르다. Fast-forward는 새 커밋을 전혀 만들지 않는다.

그래도 fast-forward를 알아야 하는 이유는, 로컬에서 브랜치를 정리하거나 CI 스크립트를 짤 때 계속 마주치기 때문이다. 아래에서 함께 다룬다.

## 비교에 사용할 공통 상황

네 방식 모두 아래 상황에서 시작한다고 가정한다.

```text
main:     A --- B --- F
                \
feature:         C --- D --- E
```

- `A`, `B`: 공통 조상까지의 커밋
- `C`, `D`, `E`: 기능 브랜치에서 작업한 커밋 3개
- `F`: 내가 작업하는 동안 다른 사람이 `main`에 올린 커밋

두 브랜치가 `B` 이후로 갈라졌다(diverged). 이 상태에서 `feature`를 `main`에 합친다.

## 1. Merge Commit (3-way merge)

### 결과

```text
main:     A --- B --- F --------- M
                \                /
feature:         C --- D --- E --
```

`M`이라는 **새 머지 커밋**이 생기고, 이 커밋은 **부모가 둘**(`F`와 `E`)이다. 기존 커밋 `C`, `D`, `E`는 해시가 그대로 유지된 채 히스토리에 남는다.

### 왜 "3-way"인가

두 브랜치의 최종 상태만 비교하면 어느 쪽이 바꾼 것인지 알 수 없다. 그래서 Git은 **세 지점**을 본다.

```text
        B        ← merge base (공통 조상)
       / \
      F   E      ← 양쪽의 현재 상태
```

| 비교 | 판단 |
| --- | --- |
| `B` → `F`만 변경됨 | `F`의 내용을 채택 |
| `B` → `E`만 변경됨 | `E`의 내용을 채택 |
| 양쪽 다 같은 곳을 변경 | **충돌**. 사람이 해결 |

공통 조상을 기준으로 삼기 때문에 "누가 무엇을 바꿨는지"를 구분할 수 있다. 충돌이 났을 때 `<<<<<<< HEAD` 사이에 보이는 것이 바로 이 두 갈래다.

### 장단점

| 장점 | 단점 |
| --- | --- |
| 실제 작업 흐름이 그대로 남는다 | 브랜치가 많으면 그래프가 복잡해진다 |
| 커밋 해시가 보존된다 | 자잘한 WIP 커밋까지 `main`에 남는다 |
| 언제 무엇이 합쳐졌는지 명확하다 | `git log`가 시간순으로 뒤섞여 읽기 어렵다 |
| 되돌릴 때 머지 단위로 다룰 수 있다 | |

복잡해 보이는 그래프는 `--first-parent`로 완화할 수 있다.

```bash
git log --oneline --first-parent   # 머지 커밋만 따라가며 main의 흐름만 본다
```

## 2. Squash and Merge

### 결과

```text
main:     A --- B --- F --- S

(feature의 C, D, E는 main에 존재하지 않음)
```

`C`, `D`, `E`의 **변경 내용을 전부 합쳐 새 커밋 `S` 하나**를 만든다. 부모는 `F` 하나뿐이다.

원래 커밋 3개는 `main` 히스토리에 들어오지 않는다. PR 페이지에는 기록이 남지만, `main`의 `git log`에는 `S` 하나만 보인다.

### 예시

```text
머지 전 feature 브랜치의 커밋
  C: "로그인 폼 추가"
  D: "오타 수정"
  E: "리뷰 반영: 유효성 검사 추가"

머지 후 main
  S: "로그인 기능 구현 (#42)"
```

"오타 수정" 같은 커밋이 `main` 히스토리를 채우지 않는다는 점이 핵심이다.

### 장단점

| 장점 | 단점 |
| --- | --- |
| `main` 히스토리가 PR 단위로 깔끔해진다 | 중간 과정이 사라져 왜 그렇게 됐는지 추적이 어렵다 |
| 되돌리기가 가장 쉽다 (커밋 하나만 revert) | 큰 PR이면 커밋 하나가 지나치게 비대해진다 |
| WIP 커밋을 신경 쓰지 않고 작업할 수 있다 | `git blame`이 전부 같은 커밋을 가리킨다 |
| `git bisect`의 단위가 명확해진다 | 브랜치 재사용 시 문제가 생긴다 |

### 주의: 머지 후 브랜치를 재사용하지 말 것

Squash 머지의 가장 흔한 함정이다. `S`는 `C`, `D`, `E`와 **아무 관계가 없는 새 커밋**이라, Git이 보기에 `feature` 브랜치의 작업은 여전히 `main`에 반영되지 않은 상태다.

이 상태에서 `feature` 브랜치에 계속 커밋하고 다시 PR을 올리면, Git은 `C`, `D`, `E`를 **또 합치려 시도**하면서 대량 충돌을 만든다.

**해결책은 간단하다. 머지 후 브랜치를 삭제하고 새로 딴다.** GitHub 저장소 설정의 "Automatically delete head branches"를 켜두면 자동으로 정리된다.

## 3. Rebase and Merge

### 결과

```text
main:     A --- B --- F --- C' --- D' --- E'
```

`C`, `D`, `E`를 `F` 위에 **하나씩 다시 적용**해 `C'`, `D'`, `E'`를 만든다. 커밋은 3개 그대로지만 **부모가 바뀌었으므로 해시가 전부 달라진다.**

머지 커밋이 없어 히스토리가 완전히 선형이 된다.

### 커밋이 보존되지만 같은 커밋은 아니다

```text
C (부모: B, 해시: a1b2c3)
   ↓ 복사
C' (부모: F, 해시: 9f8e7d)   ← 내용은 같지만 다른 커밋
```

커밋 해시가 부모 해시를 포함해 계산되기 때문에, 부모가 바뀌면 해시가 바뀔 수밖에 없다.

### 장단점

| 장점 | 단점 |
| --- | --- |
| 히스토리가 선형이라 읽기 쉽다 | 원래 해시가 사라진다 |
| 커밋 단위가 보존된다 | 각 중간 커밋은 CI를 통과한 적 없을 수 있다 |
| `git bisect`가 세밀하게 동작한다 | 충돌이 커밋마다 반복될 수 있다 |
| `main`이 시간순으로 정렬된다 | 실제로 언제 합쳐졌는지 정보가 사라진다 |

"각 중간 커밋이 CI를 통과한 적 없다"는 점은 생각보다 중요하다. `C'`은 `F` 위에서 처음 만들어진 조합이라 그 상태로 빌드된 적이 없다. `git bisect`로 문제를 찾다가 실제로는 존재한 적 없는 상태에서 실패할 수 있다.

## 4. Fast-forward Merge

### 조건

`main`이 `feature`의 **조상일 때만** 가능하다. 즉 갈라진 이후 `main`에 새 커밋이 없어야 한다.

```text
합치기 전
main:     A --- B
                \
feature:         C --- D --- E

합친 후 (포인터만 이동)
main:     A --- B --- C --- D --- E
                                  ↑ main, feature 둘 다 여기를 가리킴
```

### 결과

**새 커밋이 하나도 생기지 않는다.** 브랜치 포인터를 앞으로 옮기는 것이 전부다. 앞서 본 것처럼 브랜치는 41바이트짜리 파일이므로, 그 안의 해시만 바뀐다.

### 우리의 공통 상황에서는 불가능하다

앞의 예시에는 `F`가 있어 `main`이 앞서 나갔다. 이 경우 fast-forward는 성립하지 않고 Git이 거부한다.

```bash
git merge --ff-only feature
# fatal: Not possible to fast-forward, aborting.
```

먼저 `feature`를 `main` 위로 rebase해서 조건을 만들어야 한다.

```bash
git switch feature
git rebase main          # C, D, E를 F 위로 옮김
git switch main
git merge --ff-only feature   # 이제 fast-forward 가능
```

이 조합이 GitHub의 "Rebase and merge"가 하는 일과 사실상 같다.

### 관련 옵션

```bash
git merge --ff-only <branch>   # fast-forward만 허용, 아니면 실패
git merge --no-ff <branch>     # 가능해도 항상 머지 커밋 생성
```

`--no-ff`는 "이 브랜치에서 작업했다"는 기록을 남기고 싶을 때 쓴다. GitHub의 "Create a merge commit"이 이 동작이다.

## 네 방식 한눈에 비교

```text
[원본]
main:     A --- B --- F
                \
feature:         C --- D --- E


[1. Merge Commit]              [2. Squash]
A --- B --- F ------- M        A --- B --- F --- S
       \             /
        C --- D --- E


[3. Rebase]                    [4. Fast-forward] (F가 없을 때만)
A --- B --- F --- C' --- D' --- E'    A --- B --- C --- D --- E
```

| 항목 | Merge Commit | Squash | Rebase | Fast-forward |
| --- | --- | --- | --- | --- |
| 새로 생기는 커밋 | 머지 커밋 1개 | 1개 | 원본 수만큼 (복사) | **0개** |
| 원본 커밋 보존 | 그대로 | 사라짐 | 해시 변경 | 그대로 |
| 부모가 2개인 커밋 | 있음 | 없음 | 없음 | 없음 |
| 히스토리 모양 | 갈래 | 선형 | 선형 | 선형 |
| 브랜치 작업 흔적 | 남음 | 사라짐 | 사라짐 | 사라짐 |
| 되돌리기 난이도 | 까다로움 | **가장 쉬움** | 커밋 수만큼 | 커밋 수만큼 |
| GitHub 버튼 | 있음 | 있음 | 있음 | 없음 |

## 되돌리기 관점

실무에서 방식을 고르는 데 가장 큰 영향을 주는 부분이다.

### Squash — 가장 단순하다

```bash
git revert S
```

PR 하나가 커밋 하나이므로 그 커밋만 되돌리면 끝난다.

### Merge Commit — 옵션이 필요하다

머지 커밋은 부모가 둘이라 "어느 쪽으로 되돌릴지" 지정해야 한다.

```bash
git revert -m 1 M      # 1번 부모(main 쪽)를 기준으로 되돌린다
```

여기서 **함정**이 하나 있다. 머지를 revert하면 Git은 그 브랜치가 이미 합쳐졌다고 계속 인식한다. 나중에 같은 브랜치를 다시 머지해도 **변경 사항이 돌아오지 않는다.** 되살리려면 revert한 커밋을 다시 revert해야 한다.

```bash
git revert <revert-커밋>   # 되돌린 것을 되돌린다
```

### Rebase — 커밋 수만큼

`C'`, `D'`, `E'`를 각각 되돌리거나 범위로 지정해야 한다.

```bash
git revert E' D' C'        # 역순으로
git revert F..E'           # 범위 지정
```

## 어떤 상황에 무엇을 쓸까

### Squash and Merge가 맞는 경우

- **작은 기능 PR 중심의 팀.** PR 하나 = 배포 단위 = 되돌리기 단위로 맞아떨어진다.
- **커밋 습관이 제각각인 팀.** "wip", "fix", "다시 fix" 같은 커밋이 `main`에 쌓이는 것을 막는다.
- **오픈소스 저장소.** 외부 기여자의 커밋 정리 상태를 신뢰할 수 없을 때 유용하다.
- **빠른 롤백이 중요한 서비스.** revert 한 번으로 끝난다.

가장 무난한 기본값이다. 팀 규칙이 아직 없다면 여기서 시작하는 것을 권한다.

### Merge Commit이 맞는 경우

- **장기 기능 브랜치.** 여러 명이 몇 주간 작업한 브랜치를 하나로 뭉개면 정보 손실이 크다.
- **릴리스 브랜치 운영.** `release/1.2`를 `main`과 `develop`에 합치는 것처럼, 브랜치 간 관계 자체가 의미 있을 때.
- **커밋 하나하나가 의미 있게 정리된 팀.** 리뷰에서 커밋 단위까지 관리하는 경우.
- **감사(audit) 요구가 있는 조직.** 언제 무엇이 어떤 경로로 들어왔는지 남겨야 할 때.

### Rebase and Merge가 맞는 경우

- **선형 히스토리를 원하면서 커밋 단위도 지키고 싶은 팀.**
- **작성자가 커밋을 의미 단위로 정리해서 올리는 문화**가 있을 때. 그렇지 않으면 정돈되지 않은 커밋이 그대로 `main`에 선형으로 쌓여 squash보다 나쁜 결과가 된다.
- 다만 **중간 커밋이 검증된 적 없다는 점**을 팀이 이해하고 있어야 한다.

### Fast-forward가 맞는 경우

- **로컬에서 개인 브랜치를 정리할 때.** 혼자 작업한 브랜치를 깔끔히 반영한다.
- **릴리스 태그를 옮기는 자동화 스크립트.** 예상치 못한 머지 커밋을 막는다.
- **`--ff-only`를 CI 검증에 활용.** "이 브랜치가 최신 `main` 위에 있는가"를 확인하는 용도로 쓸 수 있다.

### 상황별 요약

| 상황 | 권장 |
| --- | --- |
| 작은 PR, 빠른 배포, 롤백 중시 | Squash |
| 장기 브랜치, 여러 명 협업 | Merge Commit |
| 릴리스 브랜치 병합 | Merge Commit |
| 선형 히스토리 + 정리된 커밋 문화 | Rebase |
| 오픈소스 외부 기여 | Squash |
| 핫픽스 한 줄 수정 | Squash |
| 개인 브랜치 로컬 정리 | Fast-forward |

## 저장소 설정으로 강제하기

방식은 개인이 매번 고르는 것보다 **팀 규칙으로 정해 두는 편**이 낫다.

**Settings → General → Pull Requests**에서 허용할 방식만 체크하면 나머지 버튼은 사라진다. 예를 들어 Squash만 남기면 팀원이 실수로 다른 방식을 고를 수 없다.

같은 화면에서 함께 설정할 만한 것들이 있다.

- **Squash 머지의 기본 커밋 메시지**: PR 제목과 본문을 쓰도록 지정하면 메시지 품질이 올라간다.
- **Automatically delete head branches**: squash 머지 시 브랜치 재사용 문제를 예방한다.

브랜치 보호 규칙의 **Require linear history**를 켜면 머지 커밋이 아예 거부되므로, Squash나 Rebase만 쓰도록 강제된다.

## 자주 하는 오해와 실수

### "Rebase and merge는 fast-forward다"

아니다. Fast-forward는 새 커밋을 만들지 않지만, GitHub의 rebase는 항상 커밋을 복사해 새 해시를 만든다. 결과 모양이 선형으로 같아 보일 뿐이다.

### Squash 머지 후 같은 브랜치를 계속 사용

`main`에는 원본 커밋이 없어서 Git이 아직 안 합쳐진 것으로 본다. 다음 PR에서 대량 충돌이 난다. 머지 후 브랜치를 삭제한다.

### 여러 명이 공유하는 브랜치를 rebase

rebase는 커밋 해시를 바꾸므로 다른 사람의 로컬 히스토리와 어긋난다. **공유 브랜치는 rebase하지 않는다.** 아직 push하지 않았거나 혼자 쓰는 브랜치에만 적용한다.

### 머지 커밋을 revert한 뒤 재머지 시도

변경 사항이 돌아오지 않는다. revert를 revert해야 한다.

### 팀 규칙 없이 각자 다른 방식 사용

히스토리가 일관성을 잃어 `git log`를 읽기 어려워지고, 되돌리기 절차도 매번 달라진다. 저장소 설정으로 하나만 남기는 편이 낫다.

### Squash를 쓰면서 PR을 지나치게 크게 만들기

수천 줄 변경이 커밋 하나가 되면 `git blame`과 `git bisect`가 무력해진다. Squash의 장점은 **PR이 적당히 작을 때** 나온다.

## 정리

- 같은 코드를 합치더라도 방식에 따라 히스토리 모양과 되돌리기 절차가 달라진다.
- **Merge Commit**은 부모가 둘인 커밋을 만들어 작업 흐름을 그대로 남긴다. 3-way란 공통 조상까지 세 지점을 비교한다는 뜻이다.
- **Squash**는 PR의 모든 커밋을 하나로 합친다. 히스토리가 깔끔하고 되돌리기가 가장 쉽지만 중간 과정이 사라진다.
- **Rebase**는 커밋을 대상 브랜치 위로 복사해 선형 히스토리를 만든다. 커밋 수는 유지되지만 해시가 바뀐다.
- **Fast-forward**는 새 커밋 없이 포인터만 옮기며, 대상 브랜치가 앞서 있지 않을 때만 가능하다. GitHub PR 버튼에는 없다.
- Squash 머지 후에는 브랜치를 반드시 삭제하고 새로 딴다.
- 공유 브랜치는 rebase하지 않는다.
- 방식은 개인 선택이 아니라 저장소 설정으로 통일하는 편이 낫다.

## 학습 체크리스트

- [ ] 네 방식이 각각 몇 개의 커밋을 새로 만드는지 말할 수 있는가?
- [ ] 3-way merge에서 "세 지점"이 무엇인지 설명할 수 있는가?
- [ ] Squash 머지 후 브랜치를 재사용하면 왜 충돌이 나는지 설명할 수 있는가?
- [ ] Rebase가 커밋 해시를 바꾸는 이유를 알고 있는가?
- [ ] Fast-forward가 불가능한 조건을 판단할 수 있는가?
- [ ] 머지 커밋을 되돌릴 때 `-m` 옵션이 필요한 이유를 아는가?
- [ ] 머지를 revert한 뒤 재머지가 안 되는 이유와 해결법을 아는가?
- [ ] 담당 저장소의 머지 방식이 무엇으로 설정되어 있고, 그 이유를 설명할 수 있는가?
- [ ] 공유 브랜치를 rebase하면 안 되는 이유를 설명할 수 있는가?

## 참고

- [GitHub Docs — About pull request merges](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/about-pull-request-merges)
- [GitHub Docs — Configuring commit squashing for pull requests](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/configuring-commit-squashing-for-pull-requests)
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Git Book — Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Git Book — Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
- [`git-merge` documentation](https://git-scm.com/docs/git-merge)
- [`git-revert` documentation](https://git-scm.com/docs/git-revert)
- [Linus Torvalds — How to revert a faulty merge](https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.txt)
