---
title: "Cloudflare Workers 에러를 Slack으로 받기: Incoming Webhook 알림 구성"
date: 2026-08-05
tags: [slack, webhook, cloudflare-workers, monitoring, alerting, error-handling, serverless, kv]
description: "Slack Incoming Webhook을 발급받아 Cloudflare Workers의 서버 에러를 알림으로 받는 구성을 정리하고, waitUntil·시크릿 관리·알림 중복 억제·민감정보 마스킹까지 운영 관점에서 정리한다."
---

## 학습 목적

새 프로젝트를 Cloudflare Workers로 올리면서 가장 먼저 불편했던 것은 **에러가 났는지 모른다**는 점이었다. 서버리스에는 SSH로 들어가 로그 파일을 열어볼 서버가 없고, 대시보드를 계속 들여다보고 있을 수도 없다.

그래서 에러를 **가져와서 보는(pull)** 방식 대신 **밀어서 알려주는(push)** 방식이 필요했고, 가장 적은 비용으로 도입할 수 있는 것이 Slack Incoming Webhook이었다.

이 글에서는 웹훅 발급부터 Workers에 붙이는 방법, 그리고 실제로 운영하면서 반드시 필요해지는 중복 억제와 민감정보 처리까지 정리한다.

## Incoming Webhook 발급

### 발급 절차

1. <https://api.slack.com/apps> 접속 → **Create New App** → **From scratch**
2. 앱 이름과 설치할 워크스페이스 선택
3. 좌측 메뉴에서 **Incoming Webhooks** 선택 후 활성화
4. **Add New Webhook to Workspace** 클릭
5. 알림을 받을 채널 선택 후 승인
6. 생성된 URL 복사

```text
https://hooks.slack.com/services/{워크스페이스ID}/{채널ID}/{토큰 24자}
```

### 이 URL의 성격

여기서 중요한 점이 두 가지 있다.

- **URL 자체가 인증 수단이다.** 별도의 토큰 헤더가 없고, 이 URL을 아는 사람은 누구나 해당 채널에 메시지를 보낼 수 있다. 따라서 **비밀 값으로 다뤄야 한다.**
- **채널이 고정된다.** 발급할 때 고른 채널로만 전송된다. 페이로드에 `channel` 필드를 넣어도 무시된다. 채널을 나누고 싶으면 **웹훅을 여러 개 발급**한다.

### Webhook과 Bot Token 비교

| 항목 | Incoming Webhook | Bot Token (`chat.postMessage`) |
| --- | --- | --- |
| 설정 난이도 | 낮음 | OAuth 스코프 설정 필요 |
| 채널 지정 | 발급 시 고정 | 호출할 때 동적으로 지정 |
| 스레드 답글 | 불가 | 가능 |
| 메시지 수정·삭제 | 불가 | 가능 |
| 파일 업로드 | 불가 | 가능 |
| 적합한 용도 | **단방향 알림** | 상호작용, 봇 기능 |

에러 알림처럼 **한 채널에 단방향으로 쏘기만 하면 되는 경우**에는 웹훅이 적합하다. 나중에 "에러 스레드에 후속 로그를 답글로 달고 싶다" 같은 요구가 생기면 그때 봇 토큰으로 옮기면 된다.

## URL은 시크릿으로 관리한다

가장 먼저 정해야 할 것이 저장 위치다.

```toml
# wrangler.toml — 이렇게 하면 안 된다
[vars]
SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/..."
```

`wrangler.toml`의 `[vars]`는 평문이고 저장소에 커밋된다. 웹훅 URL이 유출되면 제3자가 팀 채널에 임의의 메시지를 보낼 수 있다. **시크릿으로 등록해야 한다.**

```bash
npx wrangler secret put SLACK_WEBHOOK_URL
# 프롬프트에 URL 붙여넣기
```

로컬 개발에서는 `.dev.vars` 파일을 쓰고 `.gitignore`에 넣는다.

```text
# .dev.vars
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

```gitignore
.dev.vars
.env
```

코드에서는 `env`로 접근한다. Workers에서는 Node.js처럼 `process.env`를 쓰지 않는다는 점에 주의한다.

```javascript
export default {
  async fetch(request, env, ctx) {
    // env.SLACK_WEBHOOK_URL 로 접근
  },
};
```

## 최소 구현

```javascript
async function sendToSlack(webhookUrl, text) {
  await fetch(webhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
  });
}
```

동작은 하지만 이대로 운영에 넣으면 문제가 생긴다. 아래에서 하나씩 보완한다.

## Workers 환경에서 고려할 것

### `waitUntil`로 응답을 막지 않기

Slack 전송은 외부 네트워크 요청이라 수백 밀리초가 걸릴 수 있다. 이걸 `await`로 기다린 뒤 응답하면 **에러가 난 사용자는 더 오래 기다리게 된다.**

`ctx.waitUntil()`을 쓰면 응답을 먼저 돌려주고 알림은 백그라운드에서 마저 처리한다.

```javascript
export default {
  async fetch(request, env, ctx) {
    try {
      return await handleRequest(request, env);
    } catch (error) {
      // 응답은 즉시, 알림은 백그라운드에서
      ctx.waitUntil(notifyError(error, request, env));

      return new Response(
        JSON.stringify({ message: "일시적인 오류가 발생했습니다." }),
        { status: 500, headers: { "Content-Type": "application/json" } }
      );
    }
  },
};
```

`waitUntil` 없이 그냥 `notifyError(...)`를 호출만 하고 응답하면, Workers가 응답 직후 실행 컨텍스트를 정리하면서 **전송이 중간에 끊길 수 있다.** 반드시 `waitUntil`로 감싼다.

### 알림 실패가 요청 실패가 되면 안 된다

Slack이 죽었거나 rate limit에 걸렸다고 해서 우리 API가 함께 무너지면 안 된다. 알림 함수는 어떤 예외도 밖으로 던지지 않게 만든다.

```javascript
async function notifyError(error, request, env) {
  try {
    await sendToSlack(env.SLACK_WEBHOOK_URL, buildPayload(error, request, env));
  } catch (e) {
    // 알림 실패는 삼킨다. 대신 로그에는 남긴다.
    console.error("Slack 알림 전송 실패:", e);
  }
}
```

### 서브리퀘스트 제한

Workers는 요청 하나당 외부 요청(subrequest) 개수에 제한이 있다. 무료 플랜은 50개, 유료 플랜은 1000개다. 알림 전송도 여기에 포함되므로, **요청 하나에서 에러를 여러 번 잡아 매번 알림을 보내는 구조**는 피해야 한다. 에러는 최상위에서 한 번만 처리한다.

## 메시지에 무엇을 담을 것인가

`text`만 보내면 알림은 오지만 원인 파악에 쓸모가 없다. **알림을 보고 바로 다음 행동을 정할 수 있어야** 한다.

담아야 할 최소 정보는 다음과 같다.

| 항목 | 이유 |
| --- | --- |
| 환경 (prod / staging) | 지금 당장 대응할 일인지 판단 |
| 에러 메시지와 타입 | 무엇이 터졌는지 |
| 스택 트레이스 (앞부분) | 어디서 터졌는지 |
| 요청 메서드와 경로 | 어떤 API인지 |
| 요청 식별자 (`cf-ray`) | 나중에 로그에서 찾기 위해 |
| 발생 시각 | 배포·트래픽과 대조 |

Block Kit을 쓰면 읽기 좋게 구조화할 수 있다.

```javascript
function buildPayload(error, request, env) {
  const url = new URL(request.url);
  const rayId = request.headers.get("cf-ray") ?? "unknown";
  const environment = env.ENVIRONMENT ?? "unknown";

  // 스택은 길어서 앞부분만 자른다. Slack 블록에는 길이 제한이 있다.
  const stack = (error.stack ?? String(error)).slice(0, 1500);

  return {
    // 알림 목록과 푸시에 표시되는 요약
    text: `[${environment}] ${error.name}: ${error.message}`,
    blocks: [
      {
        type: "header",
        text: { type: "plain_text", text: `🚨 서버 에러 (${environment})` },
      },
      {
        type: "section",
        fields: [
          { type: "mrkdwn", text: `*경로*\n${request.method} ${url.pathname}` },
          { type: "mrkdwn", text: `*에러*\n${error.name}` },
          { type: "mrkdwn", text: `*Ray ID*\n\`${rayId}\`` },
          { type: "mrkdwn", text: `*시각*\n${new Date().toISOString()}` },
        ],
      },
      {
        type: "section",
        text: { type: "mrkdwn", text: `*메시지*\n\`\`\`${error.message}\`\`\`` },
      },
      {
        type: "section",
        text: { type: "mrkdwn", text: `*스택*\n\`\`\`${stack}\`\`\`` },
      },
    ],
  };
}
```

`text` 필드를 blocks와 함께 넣는 이유는, **모바일 푸시 알림과 알림 목록에는 `text`가 표시**되기 때문이다. blocks만 보내면 알림 목록에서 내용이 보이지 않는다.

## 알림 피로를 막는 것이 진짜 과제

도입 직후에는 잘 동작한다. 문제는 **에러 하나가 초당 수십 건씩 발생할 때** 생긴다. 채널이 같은 메시지로 도배되면 사람들은 그 채널을 음소거하고, 결국 알림 시스템이 없는 것과 같아진다.

Slack Incoming Webhook 자체도 초당 1건 수준으로 제한되어 있어, 폭주하면 대부분 버려진다.

### 심각도로 먼저 거른다

모든 에러를 보낼 필요가 없다.

| 상황 | 알림 |
| --- | --- |
| 4xx (잘못된 요청, 인증 실패) | 보내지 않음. 정상 동작의 일부다 |
| 5xx (서버 에러) | 보냄 |
| 외부 API 타임아웃 | 임계치를 넘을 때만 |
| 예상한 비즈니스 예외 | 보내지 않음 |

의도적으로 던지는 예외와 예상 못 한 예외를 타입으로 구분해 두면 이 판단이 쉬워진다.

### 같은 에러는 쿨다운을 둔다

에러의 지문(fingerprint)을 만들어 KV에 기록하고, 일정 시간 안에 같은 지문이 또 오면 보내지 않는다.

```javascript
async function shouldNotify(error, request, env) {
  const url = new URL(request.url);

  // 같은 종류의 에러를 하나로 묶는 키
  const fingerprint = `${error.name}:${url.pathname}:${error.message}`.slice(0, 200);
  const key = `alert:${fingerprint}`;

  const seen = await env.ALERT_KV.get(key);
  if (seen) return false;

  // 5분 동안 같은 에러는 다시 보내지 않는다 (최소값 60초)
  await env.ALERT_KV.put(key, "1", { expirationTtl: 300 });
  return true;
}
```

```javascript
async function notifyError(error, request, env) {
  try {
    if (!(await shouldNotify(error, request, env))) return;
    await sendToSlack(env.SLACK_WEBHOOK_URL, buildPayload(error, request, env));
  } catch (e) {
    console.error("Slack 알림 전송 실패:", e);
  }
}
```

여기에 한 가지 한계가 있다. **Workers KV는 최종적 일관성(eventual consistency)** 이라 전 세계 엣지에 전파되는 데 시간이 걸린다. 여러 리전에서 동시에 같은 에러가 터지면 중복 알림이 몇 건 나갈 수 있다. 정확한 중복 제거가 필요하면 Durable Objects를 써야 하지만, **알림 억제 용도로는 KV로 충분**하다. 목적이 정확한 카운팅이 아니라 도배 방지이기 때문이다.

### 억제된 건수를 알려준다

완전히 삼키면 "에러가 1건인 줄 알았는데 5000건이었다"가 된다. 쿨다운이 끝날 때 그동안 몇 건이 있었는지 함께 보내면 규모를 알 수 있다.

## 민감정보를 담지 않는다

Slack 채널은 팀 여러 명이 보고, 검색되고, 오래 남는다. 에러 알림에 그대로 실려 나가기 쉬운 값들이 있다.

- `Authorization` 헤더, 쿠키, 세션 토큰
- 요청 본문의 비밀번호, 카드 번호
- 이메일, 전화번호 같은 개인정보
- 쿼리스트링에 담긴 토큰

```javascript
const SENSITIVE_KEYS = ["authorization", "cookie", "token", "password", "secret", "apikey"];

function maskHeaders(headers) {
  const result = {};
  for (const [key, value] of headers) {
    const lower = key.toLowerCase();
    result[key] = SENSITIVE_KEYS.some((k) => lower.includes(k))
      ? "***"
      : value;
  }
  return result;
}
```

**요청 본문은 아예 보내지 않는 편이 안전하다.** 꼭 필요하면 화이트리스트 방식으로 필요한 필드만 골라 담는다. 블랙리스트는 새 필드가 추가될 때마다 빠뜨리게 된다.

에러 메시지 자체에 민감한 값이 섞여 나오는 경우도 있으므로(예: `duplicate key: user@example.com`), 전송 직전에 한 번 마스킹을 거치는 것이 안전하다.

## 테스트하기

웹훅은 코드 없이 바로 확인할 수 있다.

```bash
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"text":"알림 연동 테스트"}'
```

성공하면 응답 본문이 `ok`로 온다. 이 단계에서 확인되면 이후 문제는 코드 쪽이다.

Workers 쪽은 다음으로 확인한다.

```bash
npx wrangler dev            # 로컬 실행 (.dev.vars 사용)
npx wrangler tail           # 배포된 Worker의 실시간 로그
npx wrangler secret list    # 등록된 시크릿 이름 확인
```

일부러 에러를 던지는 경로를 하나 만들어 두면 배포 후 연동 확인이 쉽다. 다만 **운영에 남겨두지 않도록** 환경 변수로 막거나 배포 전에 제거한다.

## Slack 알림만으로 부족해지는 시점

Slack 알림은 **인지**에는 훌륭하지만 **조사**에는 약하다.

| 필요한 것 | Slack 알림 | 에러 추적 도구 |
| --- | --- | --- |
| 터진 즉시 알기 | 좋음 | 좋음 |
| 같은 에러 발생 추이 | 없음 | 그래프로 제공 |
| 영향받은 사용자 수 | 없음 | 집계 |
| 해결/무시 상태 관리 | 없음 | 이슈 단위 관리 |
| 소스맵 기반 스택 | 없음 | 원본 코드 위치 |

에러 종류가 늘어나고 "이 에러 지난주보다 늘었나?"를 묻게 되는 시점이 오면 Sentry 같은 도구를 붙이는 것이 맞다. 그때도 Slack을 버리는 게 아니라 **에러 추적 도구가 집계하고, 임계치를 넘을 때 Slack으로 알리는** 조합으로 간다.

처음부터 무거운 도구를 붙이는 대신 웹훅으로 시작하는 것은 합리적인 선택이다. 다만 **언제 갈아탈지 기준을 미리 정해두는 편**이 좋다.

## 자주 하는 실수

### 웹훅 URL을 저장소에 커밋

`wrangler.toml`의 `[vars]`나 소스 코드에 직접 넣으면 유출된다. `wrangler secret put`을 쓴다. 이미 커밋했다면 Slack 앱 설정에서 웹훅을 **삭제하고 재발급**해야 한다. 저장소 히스토리에서 지우는 것만으로는 부족하다.

참고로 GitHub은 **push protection**으로 웹훅 URL이 포함된 푸시를 차단한다. 실제로 이 글을 작성하면서 예시로 적은 URL이 실제 형식과 일치해 푸시가 거부됐다. 마지막 토큰 자리까지 그럴듯하게 채운 예시는 문서에서도 쓰지 않는 편이 좋다.

```text
차단됨: .../T00000000/B00000000/(24자리 문자열)
안전:   .../{워크스페이스ID}/{채널ID}/{토큰}
```

### `waitUntil` 없이 알림 전송

응답 직후 실행 컨텍스트가 정리되면서 전송이 끊길 수 있다.

### 알림 전송을 `await`로 기다린 뒤 응답

에러 응답이 그만큼 느려진다. 사용자 경험이 나빠진다.

### 알림 실패를 잡지 않음

Slack 장애가 API 장애로 번진다. `try/catch`로 삼킨다.

### 모든 에러를 알림으로 전송

4xx까지 보내면 채널이 잡음으로 가득 차고, 정작 중요한 5xx를 놓친다.

### 중복 억제 없이 운영

에러 하나가 폭주하면 채널이 도배되고 Slack rate limit에 걸려 대부분 유실된다.

### 스택 트레이스 전체를 그대로 전송

Slack 블록에는 길이 제한이 있어 잘리거나 전송이 실패한다. 앞부분만 자른다.

### 요청 본문을 통째로 전송

비밀번호나 개인정보가 채널에 남는다. 필요한 필드만 화이트리스트로 담는다.

## 정리

- Incoming Webhook은 URL 자체가 인증 수단이므로 시크릿으로 관리하고, 채널은 발급 시점에 고정된다.
- 단방향 알림에는 웹훅이면 충분하고, 스레드·수정·동적 채널이 필요해지면 봇 토큰으로 옮긴다.
- Workers에서는 `wrangler secret put`으로 URL을 등록하고 `env`로 접근한다. `[vars]`에 넣으면 안 된다.
- 알림 전송은 `ctx.waitUntil()`로 감싸 응답을 지연시키지 않으면서 전송이 끊기지 않게 한다.
- 알림 실패는 반드시 삼켜서 본 요청에 영향이 가지 않게 한다.
- 메시지에는 환경·경로·에러·`cf-ray`를 담고, `text` 필드를 함께 넣어야 푸시 알림에 내용이 보인다.
- 심각도 필터와 KV 기반 쿨다운으로 알림 피로를 막는 것이 도입 이후의 실제 과제다.
- 요청 본문과 인증 헤더는 마스킹하거나 아예 담지 않는다.
- 발생 추이와 상태 관리가 필요해지면 에러 추적 도구를 붙이고 Slack은 알림 채널로만 남긴다.

## 학습 체크리스트

- [ ] Incoming Webhook과 Bot Token의 차이와 선택 기준을 설명할 수 있는가?
- [ ] 웹훅 URL이 왜 비밀 값인지, 유출 시 무엇을 해야 하는지 아는가?
- [ ] Workers에서 시크릿을 등록하고 접근하는 방법을 아는가?
- [ ] `ctx.waitUntil()`이 필요한 이유를 설명할 수 있는가?
- [ ] 알림 전송 실패가 API 응답에 영향을 주지 않도록 처리했는가?
- [ ] 어떤 에러를 알림으로 보내고 어떤 것은 보내지 않을지 기준이 있는가?
- [ ] 같은 에러가 폭주할 때 채널이 도배되지 않도록 조치했는가?
- [ ] 알림에 인증 헤더나 개인정보가 실려 나가지 않는지 확인했는가?
- [ ] 에러 추적 도구로 넘어갈 기준을 정해 두었는가?

## 참고

- [Slack API — Sending messages using incoming webhooks](https://api.slack.com/messaging/webhooks)
- [Slack API — Block Kit 개요](https://api.slack.com/block-kit)
- [Slack API — Rate limits](https://api.slack.com/apis/rate-limits)
- [Cloudflare Workers — Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Cloudflare Workers — `waitUntil`](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [Cloudflare Workers — Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Cloudflare Workers KV — 작동 방식](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Cloudflare — `wrangler tail`](https://developers.cloudflare.com/workers/wrangler/commands/#tail)
