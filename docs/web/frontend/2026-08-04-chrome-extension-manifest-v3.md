---
title: "크롬 확장 프로그램 만들기: Manifest V3 구조와 실행 컨텍스트"
date: 2026-08-04
tags: [chrome-extension, manifest-v3, browser, javascript, service-worker, content-script, web-store]
description: "Manifest V3 기준으로 확장 프로그램의 구성 요소와 실행 컨텍스트를 정리하고, 메시지 패싱·저장소·권한 설계와 배포 과정을 정리한다."
---

## 학습 목적

크롬 확장 프로그램은 HTML, CSS, JavaScript만 알면 시작할 수 있다. 그런데 막상 만들어 보면 "왜 내 스크립트에서 페이지의 변수에 접근이 안 되지", "왜 전역 변수에 넣어둔 값이 사라지지" 같은 문제에 부딪힌다.

이 문제들은 문법이 아니라 **확장 프로그램이 여러 개의 분리된 실행 컨텍스트로 나뉘어 동작한다**는 구조에서 나온다. 이 글에서는 Manifest V3 기준으로 그 구조를 먼저 정리하고, 그 위에서 메시지 패싱·저장소·권한을 어떻게 설계해야 하는지 정리한다.

## Manifest V2와 V3

새 확장 프로그램은 **Manifest V3(MV3)로만 등록할 수 있다.** MV2는 단계적으로 지원이 종료되었으므로, 지금 학습한다면 MV3만 보면 된다. 다만 인터넷에 있는 오래된 예제는 대부분 MV2 기준이라 구분할 줄 알아야 한다.

| 항목 | MV2 | MV3 |
| --- | --- | --- |
| 백그라운드 | 항상 떠 있는 background page | **이벤트 기반 service worker** |
| 원격 코드 실행 | 가능 (CDN 스크립트 로드 등) | **금지**, 모든 코드가 패키지에 포함되어야 함 |
| `eval`, 인라인 스크립트 | 제한적으로 가능 | 금지 |
| 네트워크 요청 차단 | `webRequest`로 직접 차단 | `declarativeNetRequest`로 규칙 선언 |
| 아이콘 API | `browser_action` / `page_action` | `action`으로 통합 |
| 호스트 권한 | `permissions`에 함께 | `host_permissions`로 분리 |

MV3의 가장 큰 변화는 **백그라운드가 항상 떠 있지 않다**는 점이다. 이것이 뒤에서 다룰 여러 제약의 원인이다.

## 구성 요소와 실행 컨텍스트

확장 프로그램은 하나의 프로그램이 아니라, **서로 다른 환경에서 도는 여러 조각의 모음**이다.

```text
┌──────────────────────────────────────────────────────────┐
│ 브라우저                                                   │
│                                                          │
│  ┌────────────────┐        ┌──────────────────────────┐  │
│  │ Popup          │        │ Service Worker            │  │
│  │ (아이콘 클릭)    │◀──────▶│ (백그라운드, 이벤트 기반)    │  │
│  │ 자체 DOM        │ 메시지  │ DOM 없음, 필요할 때만 실행   │  │
│  └────────────────┘        └──────────────────────────┘  │
│                                      ▲                    │
│                                      │ 메시지              │
│  ┌───────────────────────────────────┼──────────────────┐ │
│  │ 웹 페이지 탭                        │                  │ │
│  │  ┌──────────────┐   ┌──────────────▼──────────────┐  │ │
│  │  │ 페이지 자체    │   │ Content Script              │  │ │
│  │  │ JS 컨텍스트    │ ✕ │ (격리된 JS 컨텍스트)          │  │ │
│  │  └──────┬───────┘   └──────────────┬──────────────┘  │ │
│  │         └─────────  같은 DOM  ──────┘                 │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

| 구성 요소 | 실행 위치 | DOM 접근 | `chrome.*` API | 주 용도 |
| --- | --- | --- | --- | --- |
| **Service Worker** | 백그라운드 | 없음 | 대부분 사용 가능 | 이벤트 처리, 네트워크 요청, 상태 조율 |
| **Content Script** | 웹 페이지 안 | **페이지 DOM 접근 가능** | 일부만 (`storage`, `runtime` 등) | 페이지 읽기·조작 |
| **Popup** | 아이콘 클릭 시 뜨는 창 | 자체 DOM | 대부분 사용 가능 | 사용자 UI |
| **Options 페이지** | 확장 설정 화면 | 자체 DOM | 대부분 사용 가능 | 설정 |
| **Side Panel** | 브라우저 측면 패널 | 자체 DOM | 대부분 사용 가능 | 상주형 UI |

**Popup은 닫으면 그 순간 완전히 종료된다.** 팝업의 JavaScript 상태는 남지 않으므로, 유지해야 하는 값은 `chrome.storage`에 넣거나 service worker가 들고 있어야 한다.

## 가장 작은 확장 프로그램 만들기

파일 세 개면 동작한다.

```text
my-extension/
├── manifest.json
├── popup.html
└── popup.js
```

```json
{
  "manifest_version": 3,
  "name": "My First Extension",
  "version": "1.0.0",
  "description": "확장 프로그램 구조를 확인하기 위한 예제",
  "action": {
    "default_popup": "popup.html",
    "default_title": "클릭하면 열립니다"
  }
}
```

```html
<!-- popup.html -->
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="utf-8" />
    <style>
      body { width: 240px; padding: 12px; font-family: sans-serif; }
    </style>
  </head>
  <body>
    <button id="check">현재 탭 주소 보기</button>
    <p id="result"></p>
    <script src="popup.js"></script>
  </body>
</html>
```

```javascript
// popup.js
// HTML에 인라인 스크립트를 쓸 수 없으므로 별도 파일로 분리한다.
document.getElementById("check").addEventListener("click", async () => {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  document.getElementById("result").textContent = tab.url;
});
```

### 설치와 확인

1. 주소창에 `chrome://extensions` 입력
2. 우측 상단 **개발자 모드** 켜기
3. **압축해제된 확장 프로그램을 로드** 클릭 후 폴더 선택
4. 코드를 고쳤으면 카드의 새로고침 버튼을 누른다

`manifest.json`을 바꾸면 반드시 새로고침해야 하고, content script는 새로고침 후 **페이지도 다시 로드**해야 반영된다.

## manifest.json 주요 키

```json
{
  "manifest_version": 3,
  "name": "Example",
  "version": "1.0.0",

  "action": { "default_popup": "popup.html" },

  "background": {
    "service_worker": "background.js",
    "type": "module"
  },

  "content_scripts": [
    {
      "matches": ["https://example.com/*"],
      "js": ["content.js"],
      "css": ["content.css"],
      "run_at": "document_idle"
    }
  ],

  "permissions": ["storage", "tabs", "activeTab", "alarms"],
  "host_permissions": ["https://api.example.com/*"],

  "options_page": "options.html",
  "web_accessible_resources": [
    { "resources": ["injected.js"], "matches": ["https://example.com/*"] }
  ],

  "icons": { "16": "icon16.png", "48": "icon48.png", "128": "icon128.png" }
}
```

`run_at`은 content script 주입 시점을 정한다.

| 값 | 시점 |
| --- | --- |
| `document_start` | DOM 생성 전. 페이지 스크립트보다 먼저 실행해야 할 때 |
| `document_end` | DOM 완성 직후, 이미지 로드 전 |
| `document_idle` | 기본값. 페이지가 한가해진 시점 |

## Content Script와 격리된 세계

가장 자주 막히는 지점이다. Content script는 페이지에 주입되지만 **JavaScript 컨텍스트가 분리**되어 있다. 이를 격리된 세계(isolated world)라고 한다.

```text
공유하는 것    DOM (document, 요소, 이벤트)
공유하지 않는 것  JS 변수, 함수, 프로토타입, window 객체
```

```javascript
// 페이지 자체 스크립트
window.appConfig = { token: "abc" };
```

```javascript
// content.js
console.log(document.title);      // 정상 동작 (DOM은 공유)
console.log(window.appConfig);    // undefined (컨텍스트가 다름)
```

**왜 이렇게 되어 있는가**: 페이지가 확장의 코드를 건드리거나, 확장의 라이브러리 버전이 페이지와 충돌하는 것을 막기 위해서다. 보안과 안정성을 위한 설계다.

### 페이지 컨텍스트에 접근해야 한다면

정말 페이지의 JS 변수에 접근해야 한다면 스크립트를 **페이지 세계(main world)에 주입**해야 한다.

```javascript
// content.js — 페이지 세계에 스크립트 주입
const script = document.createElement("script");
script.src = chrome.runtime.getURL("injected.js");
script.onload = () => script.remove();
(document.head || document.documentElement).appendChild(script);

// 주입한 스크립트와는 DOM 이벤트나 postMessage로 통신한다
window.addEventListener("message", (event) => {
  if (event.source !== window) return;
  if (event.data.type === "FROM_PAGE") {
    console.log("페이지에서 받음:", event.data.payload);
  }
});
```

이때 `injected.js`를 `web_accessible_resources`에 등록해야 페이지에서 로드할 수 있다.

`chrome.scripting.executeScript`의 `world: "MAIN"` 옵션이나 manifest의 `content_scripts[].world`로도 같은 일을 할 수 있다. 다만 main world에 주입된 코드는 **`chrome.*` API를 쓸 수 없다.** 페이지 데이터를 읽어서 content script로 넘기는 역할까지만 맡긴다.

## 메시지 패싱

조각들이 분리되어 있으므로 통신은 메시지로 한다.

### 일회성 메시지

```javascript
// content.js -> service worker
const response = await chrome.runtime.sendMessage({
  type: "FETCH_DATA",
  url: "https://api.example.com/items",
});
console.log(response.data);
```

```javascript
// background.js (service worker)
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === "FETCH_DATA") {
    fetch(message.url)
      .then((res) => res.json())
      .then((data) => sendResponse({ data }));

    return true; // 비동기 응답을 하겠다는 신호. 없으면 채널이 즉시 닫힌다.
  }
});
```

`return true`를 빠뜨려서 응답이 오지 않는 것이 가장 흔한 실수다. 리스너가 동기적으로 끝나면 메시지 채널이 닫히기 때문에, 비동기로 응답하려면 반드시 명시해야 한다.

### 특정 탭으로 보내기

```javascript
// service worker -> content script
const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
await chrome.tabs.sendMessage(tab.id, { type: "HIGHLIGHT" });
```

대상 탭에 content script가 주입되어 있지 않으면 오류가 난다. `chrome://` 페이지나 웹스토어 페이지에는 주입이 불가능하다는 점도 함께 기억해야 한다.

### 지속 연결

주고받을 메시지가 많으면 포트를 연다.

```javascript
const port = chrome.runtime.connect({ name: "sync" });
port.postMessage({ type: "START" });
port.onMessage.addListener((msg) => console.log(msg));
```

## Service Worker의 함정

MV3에서 가장 많이 헤매는 부분이다. Service worker는 **이벤트가 없으면 종료된다.** 대략 30초 정도 유휴 상태면 내려가고, 다음 이벤트가 오면 다시 시작된다.

```javascript
// 잘못된 코드 — service worker가 종료되면 값이 사라진다
let counter = 0;

chrome.action.onClicked.addListener(() => {
  counter += 1;          // 다시 0부터 시작할 수 있다
  console.log(counter);
});
```

```javascript
// 올바른 코드 — 상태를 storage에 둔다
chrome.action.onClicked.addListener(async () => {
  const { counter = 0 } = await chrome.storage.session.get("counter");
  await chrome.storage.session.set({ counter: counter + 1 });
});
```

타이머도 마찬가지다. `setTimeout`이나 `setInterval`은 service worker가 종료되면 함께 사라지므로 `chrome.alarms`를 쓴다.

```javascript
chrome.alarms.create("refresh", { periodInMinutes: 30 });

chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === "refresh") {
    // 주기 작업
  }
});
```

정리하면 이렇다.

| 하고 싶은 것 | 잘못된 방법 | 올바른 방법 |
| --- | --- | --- |
| 상태 유지 | 전역 변수 | `chrome.storage` |
| 지연 실행 | `setTimeout` | `chrome.alarms` |
| 주기 실행 | `setInterval` | `chrome.alarms` |
| DOM 다루기 | service worker에서 직접 | content script나 offscreen document |

**이벤트 리스너는 반드시 최상위(top-level)에서 등록**해야 한다. 비동기 콜백 안에서 등록하면 service worker가 재시작될 때 리스너가 붙기 전에 이벤트가 지나갈 수 있다.

## 저장소 선택

```javascript
await chrome.storage.local.set({ key: "value" });
const { key } = await chrome.storage.local.get("key");
```

| 저장소 | 범위 | 대략적인 용량 | 용도 |
| --- | --- | --- | --- |
| `storage.local` | 해당 브라우저에만 | 약 10MB (`unlimitedStorage` 권한으로 확장) | 캐시, 대용량 데이터 |
| `storage.sync` | 로그인한 크롬 간 동기화 | 약 100KB, 항목당 8KB | 사용자 설정 |
| `storage.session` | 브라우저 세션 동안 메모리에만 | 약 10MB | 임시 상태, 토큰 |

`storage.sync`는 용량이 작고 쓰기 횟수 제한도 있으므로 **설정값 전용**으로 쓴다. 여기에 캐시를 넣으면 금방 한도에 걸린다.

민감한 값을 다룬다면 `storage.local`은 디스크에 평문으로 남는다는 점을 고려해야 한다. 세션 토큰은 `storage.session`이 더 낫다.

## 권한 설계

확장 프로그램은 설치 시 권한 목록을 사용자에게 보여준다. **권한이 넓을수록 설치 이탈률이 올라가고 심사도 까다로워진다.**

```json
{
  "permissions": ["storage", "activeTab"],
  "host_permissions": ["https://api.example.com/*"],
  "optional_permissions": ["bookmarks"],
  "optional_host_permissions": ["https://*/*"]
}
```

| 권한 | 사용자에게 보이는 경고 |
| --- | --- |
| `activeTab` | 없음 |
| `storage` | 없음 |
| `tabs` | 탐색 기록 읽기 |
| `<all_urls>` / `https://*/*` | **모든 사이트의 데이터 읽기 및 변경** |

`activeTab`은 사용자가 확장 아이콘을 클릭했을 때만 그 탭에 대한 접근을 임시로 허용한다. 광범위한 호스트 권한 없이도 "클릭하면 현재 페이지를 처리"하는 기능을 만들 수 있어 유용하다.

당장 필요하지 않은 권한은 `optional_permissions`에 넣고 실제로 그 기능을 쓸 때 요청한다.

```javascript
const granted = await chrome.permissions.request({
  permissions: ["bookmarks"],
});
if (!granted) return;
```

## 원격 코드 실행 금지

MV3에서는 **패키지에 포함되지 않은 코드를 실행할 수 없다.**

- CDN에서 `<script src="https://cdn...">`로 라이브러리를 불러올 수 없다
- `eval()`, `new Function()`을 쓸 수 없다
- HTML에 인라인 `<script>`나 `onclick="..."` 속성을 쓸 수 없다

라이브러리는 `npm`으로 설치해 번들에 포함시켜야 한다. 서버에서 **데이터**를 받아오는 것은 문제없지만, 받아온 것을 코드로 실행하는 것이 금지된다.

## 개발과 디버깅

컨텍스트마다 콘솔이 다르다는 점이 처음에 헷갈린다.

| 대상 | 확인 방법 |
| --- | --- |
| Service Worker | `chrome://extensions` → 해당 확장의 **service worker** 링크 클릭 |
| Content Script | 대상 웹페이지에서 F12 → Console (컨텍스트 선택기에서 확장 선택 가능) |
| Popup | 팝업에서 우클릭 → 검사 |
| Options 페이지 | 옵션 화면에서 F12 |

```javascript
// 저장소 내용을 통째로 확인할 때 유용하다
chrome.storage.local.get(null).then(console.log);
```

`chrome://extensions`의 **오류** 버튼에 manifest 문법 오류나 로드 실패가 모여 표시된다. 확장이 아예 동작하지 않으면 여기부터 본다.

## 빌드 도구와 TypeScript

파일 몇 개 수준을 넘어가면 번들러를 쓰는 편이 낫다.

```bash
npm create vite@latest my-extension
npm install -D @crxjs/vite-plugin @types/chrome
```

```javascript
// vite.config.js
import { defineConfig } from "vite";
import { crx } from "@crxjs/vite-plugin";
import manifest from "./manifest.json";

export default defineConfig({
  plugins: [crx({ manifest })],
});
```

`@types/chrome`를 설치하면 `chrome.*` API에 타입이 붙어 오타와 잘못된 인자를 컴파일 시점에 잡을 수 있다. API 표면이 넓고 콜백/Promise 형태가 섞여 있어 타입 도움이 크다.

## 배포

1. **개발자 등록**: Chrome Web Store 개발자 대시보드에서 일회성 등록비를 결제한다.
2. **패키징**: 확장 폴더를 zip으로 압축한다. `node_modules`나 소스맵은 제외한다.
3. **스토어 등록 정보**: 아이콘(128px), 스크린샷, 설명, 카테고리를 준비한다.
4. **개인정보 처리방침**: 사용자 데이터를 다루면 필수다. 권한을 왜 요청하는지 명확히 적어야 심사가 수월하다.
5. **심사**: 권한 범위가 넓을수록 오래 걸린다.
6. **업데이트**: `manifest.json`의 `version`을 올려 새 패키지를 업로드하면 설치된 사용자에게 자동 배포된다.

사내용이거나 공개하고 싶지 않다면 **비공개(unlisted)** 로 게시하거나, 조직 정책으로 배포할 수 있다.

## 자주 하는 실수

### 인라인 스크립트 사용

`<button onclick="handle()">`나 HTML 안의 `<script>` 블록은 CSP에 막힌다. 별도 `.js` 파일로 분리하고 `addEventListener`로 연결한다.

### content script에서 페이지 변수 접근 시도

컨텍스트가 격리되어 있어 `window.foo`가 보이지 않는다. DOM은 공유되지만 JS는 분리된다는 점을 기억한다.

### `sendMessage` 응답이 오지 않음

비동기 리스너에서 `return true`를 빠뜨린 경우다.

### service worker 전역 변수에 상태 저장

종료되면 사라진다. `chrome.storage`를 쓴다.

### manifest 수정 후 새로고침 누락

`chrome://extensions`에서 새로고침해야 반영된다. content script는 페이지도 다시 로드해야 한다.

### 처음부터 `<all_urls>` 요청

설치 경고가 커지고 심사도 까다로워진다. `activeTab`이나 필요한 도메인만으로 시작한다.

### 특수 페이지에서 동작하지 않는다고 당황하기

`chrome://`, 웹스토어, 다른 확장 페이지에는 content script를 주입할 수 없다. 정책상 막혀 있는 것이므로 우회하려 하면 안 된다.

## 정리

- 새 확장 프로그램은 Manifest V3로만 만들고, MV3의 핵심 변화는 백그라운드가 이벤트 기반 service worker라는 점이다.
- 확장은 service worker, content script, popup 등 **분리된 실행 컨텍스트의 모음**이며 통신은 메시지로 한다.
- Content script는 페이지와 DOM은 공유하지만 JS 컨텍스트는 격리된다. 페이지 변수에 접근하려면 main world 주입이 필요하다.
- Service worker는 유휴 시 종료되므로 전역 변수 대신 `chrome.storage`, `setTimeout` 대신 `chrome.alarms`를 쓴다.
- 이벤트 리스너는 최상위에서 등록해야 재시작 시 이벤트를 놓치지 않는다.
- `storage.sync`는 설정 전용, `local`은 대용량, `session`은 민감한 임시 값에 쓴다.
- 권한은 최소로 시작하고 `activeTab`과 `optional_permissions`를 활용한다.
- 원격 코드 실행이 금지되므로 라이브러리는 번들에 포함시킨다.

## 학습 체크리스트

- [ ] service worker, content script, popup이 각각 무엇을 할 수 있는지 구분할 수 있는가?
- [ ] content script가 페이지 변수에 접근하지 못하는 이유를 설명할 수 있는가?
- [ ] 비동기 메시지 응답에 `return true`가 필요한 이유를 아는가?
- [ ] service worker가 종료된 뒤에도 유지되어야 하는 값을 어디에 둘지 판단할 수 있는가?
- [ ] `storage.local`, `sync`, `session`을 상황에 맞게 선택할 수 있는가?
- [ ] `activeTab`으로 대체 가능한 기능에 광범위한 호스트 권한을 요청하고 있지 않은가?
- [ ] 각 컨텍스트의 콘솔을 어디서 여는지 아는가?
- [ ] 원격 코드 실행 금지가 라이브러리 사용에 어떤 영향을 주는지 아는가?
- [ ] 가장 작은 확장 프로그램을 직접 만들어 로드해 봤는가?

## 참고

- [Chrome for Developers — Extensions 개요](https://developer.chrome.com/docs/extensions)
- [Chrome for Developers — Manifest V3 마이그레이션](https://developer.chrome.com/docs/extensions/develop/migrate)
- [Chrome for Developers — Extension service worker 기초](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers)
- [Chrome for Developers — Content scripts](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)
- [Chrome for Developers — Message passing](https://developer.chrome.com/docs/extensions/develop/concepts/messaging)
- [Chrome for Developers — Declare permissions](https://developer.chrome.com/docs/extensions/develop/concepts/declare-permissions)
- [Chrome for Developers — `chrome.storage` API](https://developer.chrome.com/docs/extensions/reference/api/storage)
- [Chrome Web Store — 게시 절차](https://developer.chrome.com/docs/webstore/publish)
