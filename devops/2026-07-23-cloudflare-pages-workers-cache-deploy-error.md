---
layout: post
title: "Cloudflare Pages와 Workers 배포 중 캐시로 인한 오류"
date: 2026-07-23
categories: [devops]
tags: [cloudflare, pages, workers, react, deploy, cache]
description: "React 프론트엔드를 Cloudflare Pages와 Workers에 배포할 때 캐시 때문에 간헐적으로 발생한 빌드 및 배포 오류를 정리한다."
---

## 문제 상황

React로 만든 프론트엔드를 Cloudflare Pages에 빌드하고, 필요한 경우 Workers와 함께 배포하는 과정에서 가끔 빌드 또는 배포 오류가 발생했다.

- 작업 환경: React 프론트엔드, Cloudflare Pages, Cloudflare Workers
- 실행한 작업: Git push 이후 Pages 자동 빌드 또는 Wrangler 기반 Worker 배포
- 기대한 결과: 최신 코드 기준으로 빌드와 배포가 정상 완료
- 실제 결과: 동일한 코드처럼 보이는데도 특정 시점에만 빌드 실패, 오래된 결과물 반영, 정적 파일 불일치 문제가 발생

## 대표 증상

정확한 에러 메시지는 상황마다 다를 수 있지만, 캐시 문제일 때는 보통 아래 현상이 같이 나타난다.

- 로컬에서는 `npm run build`가 통과하지만 Cloudflare Pages 빌드에서만 실패한다.
- 이전 배포의 파일이나 의존성이 남아 있는 것처럼 동작한다.
- React 빌드 결과물의 JS/CSS chunk가 맞지 않아 화면이 깨지거나 흰 화면이 나온다.
- 배포 후에도 브라우저 또는 Cloudflare CDN에서 이전 파일을 계속 내려준다.
- Worker가 최신 응답 대신 이전에 캐싱된 응답을 반환한다.

## 원인

이번 현상의 핵심 원인은 코드 자체보다 배포 과정에서 사용되는 캐시였다.

Cloudflare 환경에서는 캐시가 하나만 있는 것이 아니다. 문제를 볼 때는 아래 캐시를 구분해서 봐야 한다.

- Pages build cache: Pages가 빌드 속도를 높이기 위해 패키지 매니저 캐시나 프레임워크 빌드 캐시를 재사용한다.
- CDN cache: 배포된 정적 파일이나 응답이 Cloudflare 엣지에 캐싱될 수 있다.
- Browser cache: 사용자의 브라우저가 이전 JS/CSS 파일을 들고 있을 수 있다.
- Worker cache: Worker 코드에서 Cache API, `Cache-Control`, Workers cache 설정을 사용하는 경우 이전 응답이 유지될 수 있다.

React 프로젝트에서는 특히 의존성, lockfile, build output, 정적 asset 이름이 바뀌는 과정에서 캐시가 꼬이면 간헐적인 문제처럼 보일 수 있다.

## 확인해야 할 내용

다음에 같은 문제가 발생하면 아래 정보를 먼저 모아야 한다.

- Cloudflare Pages 빌드 로그의 정확한 에러 메시지
- 사용한 패키지 매니저: `npm`, `yarn`, `pnpm`, `bun`
- `package.json`과 lockfile 변경 여부
- Cloudflare Pages의 build command: 예: `npm run build`
- Cloudflare Pages의 build output directory: React/Vite 기준 보통 `dist`
- Node.js 버전과 Cloudflare Pages 빌드 환경 변수
- Pages build cache 활성화 여부
- Worker 배포 방식: dashboard, Git 연동, `wrangler deploy`
- `wrangler.toml` 또는 `wrangler.json`의 cache 관련 설정
- 배포 후 응답 헤더: `CF-Cache-Status`, `Cache-Control`
- 문제가 발생한 URL이 정적 파일인지, API 응답인지, Worker 응답인지

## 해결 방법

우선 캐시 종류를 나눠서 하나씩 확인한다.

1. 로컬 빌드 확인

```bash
npm install
npm run build
```

로컬 빌드가 실패하면 Cloudflare 캐시 문제가 아니라 프로젝트 빌드 설정이나 의존성 문제일 가능성이 높다.

2. Pages build cache 삭제

Cloudflare dashboard에서 Pages 프로젝트로 이동한 뒤 아래 경로에서 build cache를 삭제한다.

```text
Workers & Pages > Pages project > Settings > Build > Build cache > Clear Cache
```

이후 다시 배포해서 같은 오류가 반복되는지 확인한다.

3. CDN cache 확인

최신 배포가 되었는데도 브라우저에서 이전 결과가 보이면 Cloudflare CDN cache를 확인한다.

```bash
curl -I https://example.com
curl -I https://example.com/assets/example.js
```

응답 헤더에서 `CF-Cache-Status`, `Cache-Control` 값을 확인한다.

4. Worker cache 확인

Worker가 응답을 캐싱하고 있다면 Worker 설정과 응답 헤더를 확인한다.

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2026-07-23"

[cache]
enabled = false
```

디버깅 중에는 Worker cache를 끄거나, 캐시된 응답을 purge한 뒤 다시 확인한다. 단, cache 설정을 끄는 것만으로 이미 저장된 캐시가 즉시 삭제되는 것은 아니므로 필요한 경우 별도 purge가 필요하다.

5. 브라우저 캐시 확인

운영 배포 확인 시에는 시크릿 창, 강력 새로고침, 다른 브라우저를 사용해서 브라우저 캐시 영향을 분리한다.

## 재발 방지

React 정적 배포에서는 아래 기준을 지키는 것이 좋다.

- `index.html`은 짧게 캐싱하거나 재검증되도록 관리한다.
- hash가 붙은 JS/CSS asset은 길게 캐싱할 수 있다.
- 배포 오류가 발생하면 먼저 Pages build cache를 clear한 뒤 재배포한다.
- lockfile이 크게 바뀐 경우 캐시 재사용 여부를 의심한다.
- Worker에서 정적 파일이나 API 응답을 캐싱한다면 `Cache-Control` 정책을 명확히 둔다.
- 배포 확인 시 `curl -I`로 실제 응답 헤더를 확인한다.

## 결과

캐시를 삭제한 뒤 다시 빌드하고 배포하면 최신 코드 기준으로 정상 반영된다.

이번 문제는 React 코드 오류라기보다 Cloudflare Pages/Workers 배포 과정에서 이전 빌드 캐시 또는 응답 캐시가 재사용되면서 발생한 문제로 정리할 수 있다.

## 배운 점

- Cloudflare에서 말하는 캐시는 build cache, CDN cache, Worker cache, browser cache로 나눠서 봐야 한다.
- "로컬에서는 되는데 배포에서만 실패"하면 Pages build cache를 먼저 의심할 수 있다.
- "배포는 성공했는데 화면이 이전 상태"라면 CDN cache 또는 browser cache를 먼저 확인한다.
- Worker가 개입된 요청은 Worker cache 설정과 응답 헤더까지 함께 봐야 한다.

## 참고

- [Cloudflare Pages Build caching](https://developers.cloudflare.com/pages/configuration/build-caching/)
- [Cloudflare Pages React guide](https://developers.cloudflare.com/pages/framework-guides/deploy-a-react-site/)
- [Cloudflare Pages Build configuration](https://developers.cloudflare.com/pages/configuration/build-configuration/)
- [Cloudflare Workers cache configuration](https://developers.cloudflare.com/workers/cache/configuration/)
- [Cloudflare Purge everything](https://developers.cloudflare.com/cache/how-to/purge-cache/purge-everything/)
