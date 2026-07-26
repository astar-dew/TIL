---
layout: post
title: "Expo Metro Bundler 오류: NativeWind Babel 설정"
date: 2026-07-21
categories: [app]
tags: [expo, react-native, metro, babel, nativewind]
description: "NativeWind를 Babel plugin으로 잘못 등록해 발생한 Metro 번들 오류의 원인과 해결 과정"
---

Expo 앱에서 `npm audit fix` 실행 후 Metro 번들 오류가 발생했다. 처음에는 패키지 변경을 의심했지만, 실제 원인은 모바일 앱의 Babel 설정과 NativeWind 구성 방식이었다.

## 개발 환경

- Expo 기반 React Native 앱
- 모노레포 내 앱 경로: `apps/mobile`
- Metro Bundler, Babel, NativeWind 사용
- npm 사용
- 정확한 Expo SDK와 NativeWind 버전은 별도로 확인 필요

## 문제 상황

`npm audit fix` 실행 후 `package-lock.json`이 크게 변경되었고, Expo 앱을 실행하는 과정에서 Metro 번들이 실패했다.

실제 오류는 다음과 같았다.

```text
[BABEL] node_modules/expo/AppEntry.js:
.plugins is not a valid Plugin property
```

오류에 `expo/AppEntry.js`가 표시되지만 Expo 진입 파일 자체의 문제는 아니다. Metro가 진입 파일을 Babel로 변환하는 과정에서 잘못된 Babel 설정을 읽으면서 실패한 것이다.

## 기존 설정

`apps/mobile/babel.config.js`에서 NativeWind를 Babel plugin으로 등록하고 있었다.

```js
module.exports = function babelConfig(api) {
  api.cache(true)

  return {
    presets: ['babel-preset-expo'],
    plugins: ['nativewind/babel'],
  }
}
```

## 원인

이 프로젝트에 설치된 `nativewind/babel`은 일반 Babel plugin이 아니라 Babel preset으로 사용해야 한다.

Babel plugin 자리에는 보통 변환을 수행하는 plugin 객체가 들어가야 한다. 그런데 `nativewind/babel`이 반환한 값에는 내부 `plugins` 설정이 포함되어 있어, 이를 `plugins` 배열에 넣으면 Babel은 plugin이 가질 수 없는 `plugins` 속성을 받았다고 판단한다.

```text
plugins 배열
└── nativewind/babel
    └── { plugins: [...] }
```

그 결과 `.plugins is not a valid Plugin property` 오류가 발생했다.

## 해결 방법

`nativewind/babel`을 `plugins`에서 제거하고 `presets`에 등록한다.

```js
module.exports = function babelConfig(api) {
  api.cache(true)

  return {
    presets: ['babel-preset-expo', 'nativewind/babel'],
  }
}
```

설정 변경 후 기존 Babel과 Metro 변환 결과가 남지 않도록 캐시를 초기화해서 실행한다.

```bash
cd apps/mobile
npm run start -- --clear
```

핵심은 캐시 삭제 자체가 아니라 Babel 설정을 먼저 수정하는 것이다. 잘못된 설정을 유지한 채 캐시만 삭제하면 같은 오류가 다시 발생한다.

## 확인 결과

확인한 변경 및 검증 내용은 다음과 같다.

- `apps/mobile/package.json`은 변경되지 않았다.
- `apps/mobile/package-lock.json`은 `npm audit fix`로 많은 항목이 변경되었다.
- `npm run typecheck`는 통과했다.
- Babel 설정 수정 후 Metro를 `--clear` 옵션으로 다시 실행해야 한다.

TypeScript 검사가 통과하더라도 Metro 번들이 성공한다는 의미는 아니다. `typecheck`는 Babel plugin과 preset 구성이 유효한지 검사하지 않으므로, 최종 확인은 앱을 다시 실행해 번들 완료 여부를 확인해야 한다.

## `npm audit fix`와의 관계

이번 오류의 직접 원인은 `npm audit fix` 명령 자체가 아니라 잘못된 Babel 설정이다.

다만 `npm audit fix`가 `package-lock.json`의 하위 의존성을 변경하면서 기존에 드러나지 않던 설정 문제가 나타났을 가능성은 있다. 따라서 명령 실행 시점과 오류 발생 시점이 같더라도 실제 오류 메시지를 기준으로 원인을 구분해야 한다.

`npm audit fix --force`는 아직 실행하지 않는 것이 좋다. `--force`는 현재 의존성 범위를 벗어난 주요 버전 변경을 허용하므로 Expo 또는 React Native 생태계의 호환성을 깨뜨릴 수 있다.

먼저 다음 순서로 대응한다.

1. Babel 설정을 수정한다.
2. Metro 캐시를 초기화하고 앱을 실행한다.
3. 앱 실행과 주요 화면을 확인한다.
4. `package-lock.json` 변경 내용을 검토한다.
5. 남은 보안 취약점은 영향 범위와 업그레이드 비용을 확인한 뒤 별도로 처리한다.

## NativeWind 버전 주의사항

NativeWind 설정은 주요 버전에 따라 다르다.

- NativeWind v4 계열은 `nativewind/babel`을 Babel preset으로 구성한다.
- NativeWind v5 계열은 마이그레이션 과정에서 Babel 설정에서 NativeWind를 제거하도록 안내한다.

따라서 다른 프로젝트에 같은 수정 사항을 그대로 적용하기 전에 설치된 버전을 확인해야 한다.

```bash
npm ls nativewind
```

버전 확인 후 해당 버전의 공식 설치 문서에 맞춰 `babel.config.js`와 `metro.config.js`를 구성한다.

## 정리

- Metro 오류처럼 보여도 실제 원인은 Babel 설정일 수 있다.
- 오류 메시지의 `AppEntry.js`는 오류가 발견된 지점이며 원인 파일이 아닐 수 있다.
- 이 프로젝트에서는 `nativewind/babel`을 plugin이 아닌 preset으로 등록해야 했다.
- Babel 설정을 수정한 뒤 Metro 캐시를 초기화해야 한다.
- `typecheck` 통과와 Metro 번들 성공은 서로 다른 검증이다.
- `npm audit fix --force`는 Expo 호환성을 확인하기 전에 실행하지 않는다.

## 체크리스트

- 최초 Babel 오류 메시지를 확인했는가?
- `nativewind/babel`이 설치된 버전에 맞는 위치에 등록되어 있는가?
- 설정 변경 후 Metro 캐시를 초기화했는가?
- TypeScript 검사뿐 아니라 실제 Metro 번들도 확인했는가?
- `package.json`과 `package-lock.json` 변경을 구분해서 확인했는가?
- `npm audit fix --force` 실행 전에 주요 버전 변경 여부를 검토했는가?

## 참고 자료

- [NativeWind Expo 설치](https://www.nativewind.dev/docs/getting-started/installation)
- [NativeWind v5 마이그레이션](https://www.nativewind.dev/v5/guides/migrate-from-v4)
- [Expo Metro Bundler 설정](https://docs.expo.dev/guides/customizing-metro/)
- [Expo 캐시 초기화](https://docs.expo.dev/troubleshooting/clear-cache-macos-linux/)
