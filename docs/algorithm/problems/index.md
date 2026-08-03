# Problems

코딩테스트 문제 풀이와 접근 과정을 기록합니다.

정답 코드만 남기지 않고, **어떻게 접근했고 왜 그 방법을 선택했는지**를 함께 적습니다. 나중에 다시 볼 때 필요한 것은 코드가 아니라 사고 과정입니다.

## 폴더 구성

문제 유형별로 폴더를 나눕니다. 한 폴더에 수백 개가 쌓이면 찾기 어려워지기 때문입니다.

```text
problems/
├── dp/
├── graph/
├── binary-search/
├── greedy/
├── two-pointer/
└── string/
```

폴더는 해당 유형의 첫 글을 작성할 때 만듭니다. 새 폴더에는 `.pages` 파일을 함께 두어 사이드바에 표시할 이름을 지정합니다.

```yaml
# problems/dp/.pages
title: DP
order: asc
sort_type: natural
nav:
  - index.md
  - ...
```

## 파일명 규칙

`<플랫폼>-<문제번호>-<문제이름>.md` 형식을 사용합니다. 개념 글과 달리 날짜를 붙이지 않습니다. 다시 찾을 때 기준이 되는 것은 작성일이 아니라 문제 번호이기 때문입니다.

```text
boj-1697-숨바꼭질.md
boj-11053-가장-긴-증가하는-부분-수열.md
programmers-42586-기능개발.md
leetcode-53-maximum-subarray.md
```

번호순 정렬을 위해 `sort_type: natural`을 사용하므로 `boj-2`가 `boj-10`보다 앞에 옵니다.

## 태그 규칙

폴더는 하나만 고를 수 있지만 태그는 여러 개를 달 수 있습니다. 한 문제가 여러 유형에 걸치는 경우가 많으므로 태그를 꼭 붙입니다.

```yaml
tags: [boj, dp, binary-search, gold]
```

- 플랫폼: `boj`, `programmers`, `leetcode`
- 유형: `dp`, `graph`, `bfs`, `dfs`, `binary-search`, `greedy` 등
- 난이도: `silver`, `gold`, `platinum` 또는 `level-2`

`_templates/problem-solving.md`를 복사해 시작합니다.

## Notes

아직 작성한 글이 없습니다.
