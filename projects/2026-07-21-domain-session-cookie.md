# Domain Session Cookie

프로젝트를 진행하면서 `domain.com`과 `www.domain.com`을 함께 사용할 때 로그인 세션이 따로 관리될 수 있다는 점을 알게 되었다.

## 알게 된 내용

브라우저는 `domain.com`과 `www.domain.com`을 서로 다른 호스트로 인식한다.

그래서 로그인 세션을 쿠키로 관리하는 경우, 쿠키 설정에 따라 두 주소에서 로그인 상태가 다르게 보일 수 있다.

예를 들어 `domain.com`에서 로그인했는데 `www.domain.com`으로 접속하면 로그인이 풀린 것처럼 보일 수 있다.

## 원인

쿠키는 기본적으로 쿠키를 발급한 호스트에만 적용된다.

```http
Set-Cookie: session=abc123; Path=/; Secure; HttpOnly
```

위처럼 `Domain` 속성이 없는 쿠키는 발급된 호스트 기준으로만 전송된다.

즉, 아래 두 주소는 쿠키를 따로 가질 수 있다.

```text
domain.com
www.domain.com
```

## 세션을 공유하는 방법

두 주소에서 같은 로그인 세션을 사용하려면 쿠키의 `Domain` 속성을 상위 도메인으로 설정할 수 있다.

```http
Set-Cookie: session=abc123; Domain=domain.com; Path=/; Secure; HttpOnly; SameSite=Lax
```

이렇게 설정하면 다음과 같은 하위 도메인에서 같은 쿠키를 공유할 수 있다.

```text
domain.com
www.domain.com
api.domain.com
```

## 더 단순한 해결 방법

일반적으로는 `domain.com`과 `www.domain.com`을 모두 서비스하지 않고, 하나를 대표 도메인으로 정하는 방식이 더 깔끔하다.

예를 들어:

```text
www.domain.com -> domain.com
```

또는:

```text
domain.com -> www.domain.com
```

처럼 한쪽으로 `301 Redirect`를 설정한다.

이렇게 하면 로그인 세션, OAuth 콜백 URL, CORS 설정 등을 한 도메인 기준으로 관리할 수 있다.

## Cloudflare 환경에서 설정하기

아래는 `domain.com`을 대표 도메인으로 사용하고 `www.domain.com`을 리다이렉트하는 예시다.

### 1. DNS 레코드 설정

Cloudflare의 **DNS > Records**에서 두 호스트가 모두 등록되어 있어야 한다.

| Type | Name | Target | Proxy status |
| --- | --- | --- | --- |
| `A`, `AAAA` 또는 `CNAME` | `@` | 실제 호스팅 서버 주소 | Proxied |
| `CNAME` | `www` | `domain.com` | Proxied |

실제 Target 값은 GitHub Pages, Vercel, AWS 등 사용하는 호스팅 서비스의 안내에 맞게 설정한다. Cloudflare의 Redirect Rules가 요청을 처리하려면 리다이렉트 대상이 되는 `www` 레코드도 **Proxied** 상태여야 한다.

### 2. SSL/TLS 설정

로그인 정보를 다루는 서비스라면 Cloudflare와 원본 서버 사이도 암호화되어야 한다.

1. **SSL/TLS > Overview**에서 암호화 모드를 `Full (strict)`로 설정한다.
2. 원본 서버에 유효한 인증서가 설치되어 있는지 확인한다.
3. **SSL/TLS > Edge Certificates**에서 `Always Use HTTPS`를 활성화한다.

`Flexible` 모드는 Cloudflare와 원본 서버 사이를 HTTP로 연결하므로 로그인 서비스에는 적합하지 않다. 원본 서버가 HTTP 요청을 HTTPS로 다시 보내는 설정과 충돌하면 리다이렉트가 반복될 수도 있다.

### 3. 대표 도메인으로 리다이렉트

Cloudflare의 **Rules > Redirect Rules**에서 Single Redirect를 생성한다.

```text
Rule name: Redirect www to apex
Request URL: http*://www.domain.com/*
Target URL: https://domain.com/${2}
Status code: 301
Preserve query string: Enabled
```

이 규칙은 경로와 쿼리 문자열을 유지한다.

```text
http://www.domain.com/posts/1?from=login
-> https://domain.com/posts/1?from=login
```

기존 서비스가 안정적으로 운영 중인지 먼저 확인해야 한다면 초기에는 `302`로 테스트하고, 설정이 확정된 뒤 `301`로 변경할 수 있다. 브라우저와 검색 엔진이 `301`을 오래 캐시할 수 있기 때문이다.

### 4. 애플리케이션 쿠키 설정

모든 요청을 `domain.com`으로 리다이렉트한다면 세션 쿠키에 `Domain`을 지정하지 않는 host-only 쿠키가 더 제한적이고 단순하다.

```http
Set-Cookie: session=abc123; Path=/; Secure; HttpOnly; SameSite=Lax
```

반대로 `domain.com`, `www.domain.com`, `api.domain.com`에서 세션을 실제로 공유해야 한다면 다음과 같이 상위 도메인을 지정한다.

```http
Set-Cookie: session=abc123; Domain=domain.com; Path=/; Secure; HttpOnly; SameSite=Lax
```

Cloudflare의 DNS 또는 리다이렉트 설정이 애플리케이션의 `Set-Cookie` 속성을 자동으로 바꾸지는 않는다. 쿠키 범위는 백엔드나 인증 서버에서 직접 설정해야 한다.

### 5. 동작 확인

```bash
curl -I "http://www.domain.com/posts/1?from=login"
curl -I "https://www.domain.com/posts/1?from=login"
```

응답의 `Location`이 `https://domain.com/posts/1?from=login`인지 확인한다. 로그인 후에는 브라우저 개발자 도구의 **Application > Cookies**에서 쿠키의 Domain, Secure, HttpOnly, SameSite 값을 확인한다.

## 정리

- `domain.com`과 `www.domain.com`은 서로 다른 호스트이다.
- 쿠키의 `Domain` 설정이 없으면 로그인 세션이 분리될 수 있다.
- 세션 공유가 필요하면 `Domain=domain.com`을 설정한다.
- 가능하면 대표 도메인을 하나 정하고 나머지는 리다이렉트한다.
- HTTPS 인증서, OAuth Redirect URI, CORS 설정도 도메인 기준을 맞춰야 한다.

## 프로젝트 적용 시 체크리스트

- 대표 도메인을 정했는가?
- 다른 도메인에서 대표 도메인으로 리다이렉트되는가?
- 로그인 쿠키의 `Domain`, `Secure`, `HttpOnly`, `SameSite` 설정이 적절한가?
- OAuth Redirect URI가 대표 도메인 기준으로 등록되어 있는가?
- 프론트엔드와 백엔드의 CORS 설정이 실제 운영 도메인과 일치하는가?
- Cloudflare에서 `www` DNS 레코드가 Proxied 상태인가?
- SSL/TLS 모드가 `Full (strict)`이고 원본 인증서가 유효한가?
- Redirect Rule이 경로와 쿼리 문자열을 유지하는가?
- Cloudflare와 원본 서버에 서로 충돌하는 리다이렉트 규칙이 없는가?

## 참고 자료

- [Cloudflare DNS 레코드 관리](https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/)
- [Cloudflare Single Redirect 생성](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/create-dashboard/)
- [다른 호스트로 리다이렉트하는 예시](https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-all-different-hostname/)
- [Cloudflare Full (strict) 모드](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/)
- [Cloudflare HTTPS 적용](https://developers.cloudflare.com/ssl/edge-certificates/encrypt-visitor-traffic/)
