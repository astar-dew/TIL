# TIL

개발 공부 내용을 정리하는 개인 기술 블로그 저장소입니다.

문서 사이트: **<https://astar-dew.github.io/TIL/>**

## 목표

- 학습한 내용을 주제별로 정리합니다.
- 문제 해결 과정과 트러블슈팅 기록을 남깁니다.
- MkDocs Material로 빌드해 GitHub Pages에서 검색 가능한 문서 사이트로 제공합니다.

## 저장소 구조

```text
TIL
├── docs/                 # 사이트로 발행되는 모든 문서
│   ├── index.md          # 메인 페이지
│   ├── tags.md           # 태그 색인
│   ├── web/
│   │   ├── frontend/
│   │   ├── backend/
│   │   ├── database/
│   │   └── infra/
│   ├── app/
│   ├── ai/
│   ├── cs/
│   ├── algorithm/
│   │   ├── concepts/     # 알고리즘·자료구조 개념
│   │   └── problems/     # 문제 풀이 (유형별 하위 폴더)
│   ├── design/
│   ├── devops/
│   ├── language/
│   ├── projects/
│   └── interview/
├── _templates/           # 글 작성용 템플릿 (사이트에는 발행되지 않음)
├── mkdocs.yml            # 사이트 설정
└── requirements.txt      # 빌드 의존성
```

각 카테고리 폴더의 `index.md`는 해당 섹션의 표지 겸 목차 역할을 하고, `.pages` 파일은 사이드바에 표시될 이름과 순서를 지정합니다.

## 로컬에서 실행하기

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

mkdocs serve      # http://127.0.0.1:8000 에서 실시간 미리보기
mkdocs build --strict   # 링크 오류까지 검사하며 빌드
```

## 글 작성 규칙

- 하나의 글은 하나의 주제를 다룹니다.
- 파일명은 영문 소문자와 하이픈을 사용합니다.
- 날짜가 중요한 학습 기록은 `YYYY-MM-DD-topic.md` 형식을 사용합니다.
- 단순 개념 정리는 `topic.md` 형식을 사용할 수 있습니다.
- 새 글은 `_templates/`의 템플릿을 복사해 시작합니다.
- 문서 상단 front matter에는 `title`, `date`, `tags`, `description`을 작성합니다.

```yaml
---
title: "글 제목"
date: 2026-08-02
tags: [tag1, tag2]
description: "이 글의 핵심 내용을 한 문장으로 적는다."
---
```

`tags`에 적은 값은 [태그 색인 페이지](https://astar-dew.github.io/TIL/tags/)에 자동으로 모입니다.

## 배포

`main` 브랜치에 push하면 `.github/workflows/deploy.yml`이 사이트를 빌드해 GitHub Pages로 배포합니다.
