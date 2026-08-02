---
title: "웹과 앱 서비스의 SEO 개선 방안"
date: 2026-07-23
tags: [seo, aso, search, app-store, google-search]
description: "웹 서비스와 앱 서비스에서 검색 노출을 높이기 위해 확인해야 할 SEO, ASO, 딥링크 전략을 정리한다."
---

## 학습 목적

웹과 앱 서비스는 모두 사용자가 검색을 통해 발견할 수 있어야 한다. 하지만 웹은 검색엔진 최적화가 중심이고, 앱은 앱 스토어 최적화와 앱으로 연결되는 딥링크 전략이 함께 필요하다.

이 글에서는 웹 서비스와 앱 서비스를 나눠 검색 노출을 높이는 기준을 정리한다.

## 웹 서비스 SEO

웹 서비스는 검색엔진이 페이지를 잘 발견하고, 이해하고, 색인할 수 있어야 한다.

### 1. 페이지 구조

- 페이지마다 고유한 `title`을 둔다.
- 페이지 내용을 요약하는 `meta description`을 작성한다.
- `h1`, `h2`, `h3` heading 구조를 의미 있게 사용한다.
- URL은 짧고 의미 있게 만든다.
- 중요한 페이지는 내부 링크로 연결한다.

좋은 예시는 아래와 같다.

```html
<title>React 프로젝트 배포 방법 - TIL</title>
<meta
  name="description"
  content="React 프로젝트를 빌드하고 Cloudflare Pages에 배포하는 과정을 정리합니다."
/>
```

### 2. 검색엔진 크롤링

- `robots.txt`로 크롤링 허용/차단 범위를 명확히 한다.
- `sitemap.xml`을 제공해서 주요 페이지를 검색엔진에 알려준다.
- 로그인 이후에만 볼 수 있는 내용은 검색 노출 대상에서 제외한다.
- 삭제되거나 이동한 페이지는 적절한 HTTP status와 redirect를 설정한다.

### 3. 콘텐츠 품질

- 키워드를 반복하기보다 사용자의 질문에 답하는 내용을 작성한다.
- 제목, 본문, 이미지 설명이 같은 주제를 향하도록 구성한다.
- 얇은 내용의 페이지를 많이 만들기보다 실제로 도움이 되는 페이지를 만든다.
- 공식 문서, 실제 경험, 코드 예시를 함께 넣으면 신뢰도가 높아진다.

### 4. React 서비스에서 주의할 점

React SPA는 초기 HTML에 콘텐츠가 거의 없으면 검색엔진이 내용을 이해하기 어려울 수 있다.

개선 방향은 아래와 같다.

- 검색 노출이 중요한 페이지는 SSR 또는 SSG를 고려한다.
- 단순 SPA라면 최소한 페이지별 `title`, `description`, canonical URL을 관리한다.
- 동적으로 불러오는 핵심 콘텐츠가 검색엔진 렌더링 시점에도 보이는지 확인한다.
- 공개 페이지와 로그인 전용 페이지를 분리한다.

### 5. 성능

Google은 Core Web Vitals를 사용자 경험 신호로 본다. 특히 아래 지표를 관리해야 한다.

- LCP: 주요 콘텐츠가 빠르게 표시되는지
- INP: 사용자 입력에 빠르게 반응하는지
- CLS: 화면 레이아웃이 갑자기 밀리지 않는지

개선 방법은 아래와 같다.

- 이미지 크기 최적화
- lazy loading 적용
- 불필요한 JavaScript 줄이기
- CSS/JS 번들 크기 관리
- CDN 캐시 정책 정리

### 6. 구조화 데이터

검색결과에서 콘텐츠 의미를 더 잘 전달하려면 구조화 데이터를 사용할 수 있다.

- 블로그 글: `Article`
- FAQ: `FAQPage`
- 제품: `Product`
- 조직/서비스: `Organization`
- breadcrumb: `BreadcrumbList`

단, 실제 화면에 없는 내용을 구조화 데이터에만 넣으면 안 된다.

## 앱 서비스 SEO와 ASO

앱은 일반 웹 SEO만으로는 부족하다. 앱 스토어에서의 검색 노출과 웹에서 앱으로 연결되는 흐름을 같이 봐야 한다.

### 1. 앱 스토어 최적화

App Store와 Google Play에서는 앱의 메타데이터와 시각 자료가 다운로드 전환에 큰 영향을 준다.

확인할 항목은 아래와 같다.

- 앱 이름
- 짧은 설명 또는 부제
- 전체 설명
- 키워드 또는 카테고리
- 앱 아이콘
- 스크린샷
- 미리보기 영상
- 평점과 리뷰
- 업데이트 기록

Google Play 설명은 앱 기능을 정확하게 설명해야 하며, 키워드 반복이나 과장 표현은 피해야 한다.

### 2. 앱 상세 페이지 전환율

검색 노출이 늘어도 상세 페이지에서 설치로 이어지지 않으면 효과가 낮다.

- 첫 스크린샷에서 핵심 기능을 보여준다.
- 사용자가 얻는 이점을 짧고 명확하게 작성한다.
- 기능 나열보다 실제 사용 장면을 보여준다.
- 국가/언어별로 스토어 페이지를 현지화한다.
- App Store에서는 product page optimization으로 아이콘, 스크린샷, 앱 미리보기를 테스트한다.

### 3. 딥링크

앱 서비스는 웹 URL과 앱 화면을 연결하는 구조가 중요하다.

- iOS: Universal Links
- Android: App Links
- 공통: 웹 URL을 기준으로 앱 내부 화면과 연결

예시는 아래와 같다.

```text
https://example.com/articles/123
```

앱이 설치되어 있으면 앱의 글 상세 화면으로 이동하고, 설치되어 있지 않으면 웹 페이지가 열리도록 구성하는 것이 좋다.

### 4. 앱 인덱싱

앱 내부 콘텐츠가 검색 대상이 되어야 한다면 App Indexing 또는 딥링크 기반 노출 전략을 검토한다.

앱 인덱싱이 필요한 경우는 아래와 같다.

- 앱 안에 검색 가능한 콘텐츠가 많다.
- 웹과 앱에 같은 콘텐츠 상세 페이지가 있다.
- 사용자가 검색 결과에서 바로 앱의 특정 화면으로 들어오길 원한다.

### 5. 웹 랜딩 페이지

앱만 있어도 웹 랜딩 페이지는 필요하다.

- 앱 소개 페이지
- 주요 기능 설명
- 가격 또는 사용 방법
- FAQ
- 앱 다운로드 링크
- 개인정보처리방침
- 고객 지원 페이지

검색엔진은 앱 자체보다 웹 페이지를 더 쉽게 탐색하므로, 앱 서비스를 알리는 공개 웹 페이지를 같이 운영하는 것이 좋다.

## 서비스별 정리

| 구분 | 웹 서비스 | 앱 서비스 |
| --- | --- | --- |
| 핵심 목표 | 검색엔진 노출 | 스토어 검색 노출과 설치 전환 |
| 주요 채널 | Google, Naver 등 검색엔진 | App Store, Google Play |
| 핵심 작업 | title, description, sitemap, 콘텐츠, 성능 | 앱 이름, 설명, 스크린샷, 리뷰, 딥링크 |
| 기술 요소 | SSR/SSG, 구조화 데이터, Core Web Vitals | Universal Links, App Links, App Indexing |
| 측정 도구 | Google Search Console, Lighthouse, Analytics | App Store Connect, Google Play Console |

## 우선순위

처음부터 모든 것을 하려고 하기보다 아래 순서로 진행하는 것이 좋다.

1. 공개 페이지의 `title`, `description`, heading 정리
2. `sitemap.xml`, `robots.txt` 설정
3. React 페이지의 SSR/SSG 필요 여부 판단
4. Core Web Vitals 개선
5. 앱 스토어 설명, 스크린샷, 키워드 정리
6. 웹 URL과 앱 화면을 연결하는 딥링크 구성
7. Search Console, App Store Connect, Play Console로 성과 측정

## 정리

웹 서비스는 검색엔진이 페이지를 잘 이해하고 색인하도록 만드는 것이 핵심이다. 앱 서비스는 앱 스토어에서 발견되도록 만드는 ASO와, 웹 검색에서 앱 내부 화면으로 이어지는 딥링크 전략이 함께 필요하다.

SEO는 한 번 설정하고 끝나는 작업이 아니라 콘텐츠, 성능, 링크, 사용자 행동 데이터를 보면서 계속 개선해야 하는 운영 작업이다.

## 참고

- [Google Search Central SEO Starter Guide](https://developers.google.com/search/docs/fundamentals/seo-starter-guide)
- [Google Search Central Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals)
- [Google Search Central JavaScript SEO](https://developers.google.com/search/docs/crawling-indexing/javascript/dynamic-rendering)
- [Google Search Central Structured data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)
- [Apple App Store product page optimization](https://developer.apple.com/help/app-store-connect/create-product-page-optimization-tests/overview-of-product-page-optimization/)
- [Apple Universal Links](https://developer.apple.com/documentation/xcode/allowing-apps-and-websites-to-link-to-your-content/)
- [Google Play store listing best practices](https://support.google.com/googleplay/android-developer/answer/13393723)
- [Firebase App Indexing](https://firebase.google.com/products/app-indexing)
