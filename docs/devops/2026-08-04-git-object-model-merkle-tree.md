---
title: "Git의 내부 구조: Merkle Tree와 커밋 DAG"
date: 2026-08-04
tags: [git, github, merkle-tree, dag, version-control, internals, hash]
description: "Git이 파일을 저장하는 방식을 blob·tree·commit 객체와 Merkle Tree로 정리하고, 커밋 히스토리가 트리가 아니라 DAG인 이유와 실무에서의 결과를 정리한다."
---

## 학습 목적

"Git은 트리 구조로 되어 있다"는 말은 절반만 맞다. Git 안에는 **성격이 다른 두 가지 구조**가 있고, 이를 구분하지 못하면 rebase가 왜 해시를 전부 바꾸는지, 브랜치 100개를 만들어도 왜 용량이 늘지 않는지 설명할 수 없다.

| 대상 | 구조 | 설명 |
| --- | --- | --- |
| 한 시점의 파일·디렉터리 스냅샷 | **Merkle Tree** | 디렉터리가 트리, 파일이 잎이며 각 노드는 해시로 식별된다 |
| 커밋들의 연결 관계 | **DAG (방향 비순환 그래프)** | 트리가 아니다. 머지 커밋은 부모가 둘 이상이다 |

먼저 전제를 하나 정리하면, 이 구조는 **Git**의 것이지 GitHub의 것이 아니다. GitHub은 Git 저장소를 호스팅하고 PR·이슈·Actions를 얹은 플랫폼이고, 트리 구조는 Linus Torvalds가 2005년에 만든 Git 자체의 저장 모델이다.

이 글에서는 Git이 파일을 어떻게 저장하는지 객체 수준에서 확인하고, 그 구조에서 어떤 실무적 결과가 따라오는지 정리한다.

## Git의 네 가지 객체

Git 저장소의 `.git/objects/`에 들어가는 것은 네 종류뿐이다.

| 객체 | 저장하는 것 | 비유 |
| --- | --- | --- |
| **blob** | 파일의 **내용만** | 이름 없는 파일 덩어리 |
| **tree** | 디렉터리 목록 (이름 → blob/tree 매핑) | 디렉터리 |
| **commit** | 루트 tree 하나 + 부모 커밋 + 메타데이터 | 스냅샷 한 장 |
| **tag** | 특정 객체를 가리키는 이름표 (annotated tag) | 북마크 |

여기서 중요한 점은 **blob에는 파일 이름이 없다**는 것이다. 이름은 그 파일을 담고 있는 tree가 가지고 있다. 이 분리가 뒤에 나올 여러 특성의 출발점이다.

### 직접 열어보기

이 저장소에서 실제로 확인한 결과다.

```bash
$ git cat-file -p HEAD
tree 67cd3f84a0d84d7dd1d2be4170cac899d99a744c
parent 1c0374221585d1e090c2c1d4b361269bb9fed5d6
author Jinwoo <...> 1785776238 +0900
committer Jinwoo <...> 1785776238 +0900

Add Algorithm and Software Design categories
```

커밋은 놀랄 만큼 단순하다. **루트 tree 해시 하나, 부모 커밋 해시, 작성자 정보, 메시지**가 전부다. 커밋 자체는 파일 내용을 전혀 담고 있지 않다.

그 루트 tree를 열면 최상위 디렉터리 목록이 나온다.

```bash
$ git cat-file -p HEAD^{tree}
040000 tree 653d20edb9d1bd7e4f8d359f146dfe1795e7f832    .github
100644 blob bb550320896f216881ec94501cc1c361bb8efdde    .gitignore
100644 blob 8c9211c8ad6c24107aee7e9458683cb988cd1928    README.md
040000 tree ab64f179c696b08bc62267344787009a8b4d9c59    _templates
040000 tree c9bffc70075d10f3f10aab76c7db347a88efd86a    docs
100644 blob e8a41ee7412ba2611b1e603e9856756793cb1de3    mkdocs.yml
100644 blob 65d32f252777165c41cd2d503fb8fc4815b7d029    requirements.txt
```

디렉터리는 `tree`, 파일은 `blob`으로 이어진다. `docs`의 tree를 다시 열면 그 안의 목록이 나오고, 이 과정이 파일에 도달할 때까지 반복된다. **파일 시스템의 디렉터리 구조가 그대로 객체 그래프로 옮겨진 형태**다.

앞의 숫자는 파일 모드다.

| 모드 | 의미 |
| --- | --- |
| `100644` | 일반 파일 |
| `100755` | 실행 권한이 있는 파일 |
| `120000` | 심볼릭 링크 |
| `040000` | 디렉터리 (tree) |
| `160000` | 서브모듈 (다른 저장소의 커밋 참조) |

Git이 권한 중 실행 비트만 기록한다는 점도 여기서 드러난다. 그래서 `chmod 755`는 변경으로 잡히지만 `chmod 640`은 잡히지 않는다.

## 콘텐츠 주소 지정: 해시가 곧 주소다

Git은 객체를 저장할 때 **내용을 해시한 값을 그대로 파일 이름으로 사용**한다. 이를 콘텐츠 주소 지정(content-addressable) 방식이라 한다.

```bash
$ printf 'hello' | git hash-object --stdin
b6fc4c620b67d95f953a5c1c1230aaab5db5a1b0
```

해시 대상은 파일 내용만이 아니라 **타입과 크기를 담은 헤더가 붙은 형태**다.

```text
"blob 5\0hello"  →  SHA-1  →  b6fc4c62...
 └타입┘└크기┘└내용┘
```

저장 위치는 해시 앞 2자리를 디렉터리로 쓴다.

```text
.git/objects/b6/fc4c620b67d95f953a5c1c1230aaab5db5a1b0
```

여기서 두 가지 성질이 나온다.

1. **같은 내용은 항상 같은 해시를 갖는다.** 그래서 100개 커밋에 걸쳐 바뀌지 않은 파일은 blob 하나만 저장된다. 파일을 다른 디렉터리로 복사해도 blob은 추가되지 않고, tree에 항목만 하나 더 생긴다.
2. **내용이 1바이트라도 바뀌면 완전히 다른 해시가 된다.** 즉 해시는 내용의 지문 역할을 한다.

Git은 기본적으로 SHA-1을 쓰지만, 2017년 SHA-1 충돌 공격(SHAttered)이 발표된 뒤로는 충돌 탐지가 들어간 SHA-1DC를 사용한다. SHA-256 저장소 형식도 지원하지만 아직 기본값은 아니다.

## Merkle Tree: 해시가 위로 전파된다

핵심은 **상위 노드의 해시가 하위 노드의 해시를 포함해서 계산된다**는 점이다. tree 객체의 내용이 곧 "자식들의 이름과 해시 목록"이기 때문이다. 이런 구조를 Merkle Tree라고 한다.

```text
                commit (해시 = 트리 해시 + 부모 + 메타데이터로 계산)
                   │
                root tree
        ┌──────────┼──────────┐
     README.md    docs/     mkdocs.yml
      (blob)      (tree)      (blob)
                ┌──┴───┐
              cs/    devops/
             (tree)   (tree)
                        │
              2026-08-04-git....md
                    (blob)
```

파일 하나를 고치면 어떤 일이 벌어지는지 따라가 보면 구조가 분명해진다.

```text
docs/devops/note.md 수정
   ↓ 내용이 바뀌었으므로
blob 해시 변경
   ↓ devops tree의 목록에 적힌 해시가 바뀌므로
devops tree 해시 변경
   ↓
docs tree 해시 변경
   ↓
root tree 해시 변경
   ↓
commit 해시 변경
```

**변경이 잎에서 뿌리까지 전파된다.** 반대로 건드리지 않은 `cs/` tree의 해시는 그대로이고, 그 아래 blob들도 재사용된다.

이 성질이 주는 것이 두 가지다.

- **무결성 검증**: 커밋 해시 하나만 알면 그 시점 저장소 전체의 내용이 확정된다. 중간의 어떤 파일이든 조작되면 상위 해시가 전부 어긋나므로 즉시 드러난다. 저장소를 클론할 때 내용이 온전한지 확인할 수 있는 근거다.
- **중복 제거**: 바뀌지 않은 하위 트리는 통째로 재사용된다. 그래서 커밋마다 "전체 스냅샷"을 저장하는데도 용량이 폭증하지 않는다.

## 커밋 히스토리는 트리가 아니라 DAG다

여기가 "Git은 트리 구조"라는 말이 어긋나는 지점이다. 스냅샷 하나는 트리지만, **커밋들의 연결 관계는 트리가 아니다.**

트리는 각 노드의 부모가 하나뿐이다. 그런데 머지 커밋은 부모가 둘 이상이다.

```text
A ─── B ─── C ─────── F ─── G   (main)
             \       /
              D ─── E           (feature)

F = 머지 커밋, 부모가 C와 E 두 개
```

그래서 커밋 히스토리는 **DAG(Directed Acyclic Graph, 방향 비순환 그래프)** 다.

- **방향**: 각 커밋은 부모를 가리킨다. 과거 방향으로만 링크가 있고, 부모는 자식을 모르기 때문에 히스토리 탐색은 항상 최신에서 과거로 진행된다.
- **비순환**: 커밋 해시가 부모 해시를 포함해 계산되므로 순환이 만들어질 수 없다. 순환을 만들려면 자기 해시를 미리 알아야 하는 모순이 생긴다.

부모 수로 커밋 종류를 구분할 수 있다.

| 부모 수 | 커밋 종류 |
| --- | --- |
| 0개 | 최초 커밋 (root commit) |
| 1개 | 일반 커밋 |
| 2개 이상 | 머지 커밋 (3개 이상은 octopus merge) |

```bash
git log --graph --oneline --all     # DAG를 시각적으로 확인
```

## 브랜치와 HEAD는 그냥 포인터다

Git에서 브랜치는 무거운 개념이 아니다. **커밋 해시 하나를 담은 텍스트 파일**이다.

```bash
$ cat .git/HEAD
ref: refs/heads/main

$ wc -c < .git/refs/heads/main
41
```

41바이트다. 40자 해시 + 줄바꿈 하나. 브랜치를 만든다는 것은 이 41바이트짜리 파일을 하나 더 만드는 일이다.

```text
.git/
├── HEAD                    → "ref: refs/heads/main"
├── refs/
│   ├── heads/
│   │   ├── main            → "e9797a7..." (커밋 해시)
│   │   └── feature/login   → "4d5e6f7..."
│   └── tags/
└── objects/                → blob, tree, commit 실제 저장소
```

- **브랜치**: 커밋을 가리키는 이동 가능한 포인터. 커밋하면 이 포인터가 새 커밋으로 옮겨간다.
- **HEAD**: 지금 어느 브랜치에 있는지 가리키는 포인터. 보통 브랜치를 가리키고, 커밋 해시를 직접 가리키면 그게 **detached HEAD** 상태다.
- **태그**: 움직이지 않는 포인터.

SVN 같은 도구에서 브랜치가 디렉터리 복사였던 것과 달리, Git에서 브랜치 생성이 즉시 끝나는 이유가 이것이다. 저장소가 몇 GB든 41바이트 파일 하나만 쓰면 된다.

## 이 구조에서 따라오는 것들

내부 구조를 아는 실익은 여기에 있다.

### 커밋은 스냅샷이지만 저장은 델타로 한다

객체 모델에서 커밋은 **그 시점의 전체 스냅샷**을 가리킨다. 변경분(diff)을 저장하지 않는다. `git show`가 보여주는 diff는 부모 커밋의 트리와 비교해 **그때그때 계산한 결과**다.

다만 저장 계층은 다르다. Git은 느슨한 객체(loose object)가 쌓이면 packfile로 묶으면서 비슷한 객체끼리 델타 압축을 적용한다.

```bash
$ git count-objects -v
count: 233        # 느슨한 객체 수
size: 1048        # KB
in-pack: 0        # 팩에 들어간 객체 수
```

**모델은 스냅샷, 저장은 델타**로 나누어 이해하면 혼란이 없다.

### 히스토리를 고치면 해시가 전부 바뀐다

커밋 해시는 부모 해시를 포함해 계산된다. 따라서 과거 커밋 하나를 수정하면 그 뒤의 모든 커밋 해시가 연쇄적으로 바뀐다.

```text
수정 전:  A ─── B ─── C ─── D
수정 후:  A ─── B' ─── C' ─── D'      (B의 메시지만 고쳐도 C, D까지 새 객체)
```

`git commit --amend`, `git rebase`, `git filter-branch`가 "히스토리 재작성"이라 불리는 이유다. 정확히는 **기존 커밋을 수정하는 것이 아니라 새 커밋들을 만들고 브랜치 포인터를 옮기는 것**이다. Git 객체는 불변이라 수정 자체가 불가능하다.

이미 push한 브랜치를 rebase한 뒤 force push하면, 동료의 로컬 히스토리와 완전히 다른 커밋 열이 되는 것도 같은 이유다.

### 원래 커밋은 바로 사라지지 않는다

rebase나 reset을 하면 이전 커밋은 어느 브랜치에서도 참조되지 않는 상태(dangling)가 되지만, 객체 자체는 남아 있다. `reflog`가 HEAD의 이동 기록을 따로 보관하기 때문이다.

```bash
git reflog                    # HEAD가 거쳐 간 커밋 목록
git reset --hard <이전 해시>   # 되돌리기
```

기본 보관 기간은 도달 가능한 객체 90일, 도달 불가능한 객체 30일이다. `git gc`가 그 이후에 정리한다. **reset --hard로 날렸다고 바로 포기할 필요가 없다**는 뜻이다.

### Git은 파일 이름 변경을 기록하지 않는다

blob에는 이름이 없고 tree가 이름을 갖는다. 파일 이름만 바꾸면 blob은 그대로이고 tree의 항목만 달라진다. Git은 이 변경을 "rename"으로 기록하지 않고, **나중에 diff를 계산할 때 내용 유사도로 추정**한다.

```bash
git log --follow <파일>       # 이름 변경을 추적해서 이력 보기
git config diff.renames true  # 이름 변경 감지 활성화 (기본값)
```

앞서 이 저장소를 MkDocs로 옮기면서 `README.md`를 `index.md`로 바꿨을 때, `git log --stat`에 `ai/README.md => docs/ai/index.md`처럼 표시된 것이 이 추정의 결과다.

### 같은 파일은 브랜치가 몇 개든 한 번만 저장된다

브랜치가 10개여도 대부분의 blob과 tree를 공유한다. 브랜치별로 저장소가 복제되는 것이 아니라, **DAG에서 갈라지는 커밋만 추가**되고 나머지 객체는 재사용된다.

### 머지와 리베이스를 DAG로 이해하기

| 방식 | DAG에 생기는 일 |
| --- | --- |
| fast-forward 머지 | 새 커밋 없이 브랜치 포인터만 앞으로 이동 |
| 머지 커밋 | 부모가 둘인 커밋을 추가해 두 갈래를 합침 |
| rebase | 커밋을 새 부모 위에 **복사**해 새 해시로 다시 만듦, 원본은 dangling |
| squash 머지 | 여러 커밋 내용을 합친 **부모 하나짜리** 새 커밋 생성 |

rebase가 히스토리를 선형으로 만드는 이유는 갈래를 합치는 대신 커밋을 옮겨 붙이기 때문이고, 그 과정에서 해시가 바뀌는 것은 필연이다.

## 직접 확인해 보기

개념만 읽는 것보다 자기 저장소를 열어보는 편이 빠르다.

```bash
# 현재 커밋 객체 내용
git cat-file -p HEAD

# 객체 타입 확인 (commit / tree / blob / tag)
git cat-file -t HEAD

# 루트 트리 목록
git cat-file -p HEAD^{tree}

# 전체 파일을 재귀적으로
git ls-tree -r HEAD

# 특정 파일의 blob 해시
git rev-parse HEAD:README.md

# 그 blob의 내용
git cat-file -p $(git rev-parse HEAD:README.md)

# 저장된 객체 파일 직접 보기
find .git/objects -type f | head

# 객체 통계
git count-objects -v

# 커밋 DAG 시각화
git log --graph --oneline --all

# 부모가 여러 개인 커밋(머지)만 보기
git log --merges --oneline

# HEAD 이동 기록
git reflog
```

`git cat-file -p`로 tree를 따라 내려가다 보면 파일에 도달한다. 이 과정을 한 번 해 보면 구조가 확실히 남는다.

## 자주 하는 오해

### "Git은 변경분(diff)을 저장한다"

객체 모델에서는 각 커밋이 전체 스냅샷을 가리킨다. 화면에 보이는 diff는 계산된 결과다. 델타 압축은 packfile이라는 저장 계층의 최적화이며 모델과는 별개다.

### "커밋 히스토리는 트리 구조다"

머지 커밋의 부모가 둘 이상이므로 트리가 아니라 DAG다. 트리인 것은 **한 커밋이 가리키는 파일 스냅샷** 쪽이다.

### "GitHub이 트리 구조로 만들어졌다"

트리 구조는 Git의 저장 모델이다. GitHub은 그 위에 호스팅, 권한, PR, 코드 리뷰, Actions를 얹은 서비스다. GitLab이나 Gitea도 같은 Git 모델 위에 있다.

### "rebase는 커밋을 수정한다"

Git 객체는 불변이다. rebase는 새 커밋을 만들고 브랜치 포인터를 옮긴다. 원래 커밋은 참조를 잃을 뿐 즉시 삭제되지 않는다.

### "reset --hard 하면 커밋이 사라진다"

참조가 끊길 뿐 객체는 남아 있고, `git reflog`로 해시를 찾아 복구할 수 있다. 실제로 지워지는 시점은 `git gc`가 도달 불가능한 객체를 정리할 때다.

### "브랜치를 많이 만들면 저장소가 무거워진다"

브랜치는 41바이트 파일이고 객체는 공유된다. 늘어나는 것은 갈라진 커밋과 그로 인해 새로 생긴 blob·tree뿐이다.

## 정리

- Git 객체는 blob(내용), tree(디렉터리 목록), commit(스냅샷 + 부모), tag 네 가지다.
- blob에는 이름이 없고 tree가 이름을 갖는다. 그래서 이름 변경은 기록되지 않고 추정된다.
- 객체는 내용의 해시로 주소가 정해지므로, 같은 내용은 한 번만 저장된다.
- 상위 노드 해시가 하위 해시를 포함하는 Merkle Tree라서, 변경이 잎에서 뿌리까지 전파되고 무결성 검증이 가능하다.
- 파일 스냅샷은 트리지만 **커밋 히스토리는 부모가 여럿일 수 있는 DAG**다.
- 브랜치와 HEAD는 커밋 해시를 담은 41바이트 포인터이며, 그래서 브랜치 생성이 즉시 끝난다.
- 커밋 해시가 부모 해시를 포함하므로 히스토리 재작성은 이후 커밋 해시를 모두 바꾼다.
- 객체는 불변이다. amend와 rebase는 수정이 아니라 새 객체 생성 + 포인터 이동이다.

## 학습 체크리스트

- [ ] blob, tree, commit이 각각 무엇을 저장하는지 설명할 수 있는가?
- [ ] 파일 이름이 blob이 아니라 tree에 있는 이유와 그 결과를 설명할 수 있는가?
- [ ] `git cat-file -p HEAD`부터 파일 blob까지 직접 따라가 봤는가?
- [ ] 파일 하나를 수정했을 때 어떤 객체들의 해시가 바뀌는지 순서대로 말할 수 있는가?
- [ ] 커밋 히스토리가 트리가 아니라 DAG인 이유를 설명할 수 있는가?
- [ ] 브랜치 생성이 저장소 크기와 무관하게 즉시 끝나는 이유를 아는가?
- [ ] rebase 후 force push가 왜 동료의 히스토리와 충돌하는지 설명할 수 있는가?
- [ ] `git reset --hard`로 날린 커밋을 reflog로 복구해 봤는가?
- [ ] 커밋이 스냅샷이라는 말과 packfile의 델타 압축이 모순되지 않는 이유를 설명할 수 있는가?

## 참고

- [Git Book — Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Git Book — Git Internals: Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
- [Git Book — Git Internals: Packfiles](https://git-scm.com/book/en/v2/Git-Internals-Packfiles)
- [Git Book — Git Branching: Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [`git-cat-file` documentation](https://git-scm.com/docs/git-cat-file)
- [`git-reflog` documentation](https://git-scm.com/docs/git-reflog)
- [Git — Hash function transition to SHA-256](https://git-scm.com/docs/hash-function-transition)
- [SHAttered: SHA-1 collision attack](https://shattered.io/)
