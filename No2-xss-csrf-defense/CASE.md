## **HTTPS + HttpOnly Cookie를 적용했는데도 왜 XSS와 CSRF 공격이 가능하며, 각 공격을 막기 위해 어떤 보안 계층을 추가해야 하는가?**

### [상황]

우리 팀은 React 기반 웹 서비스를 운영하고 있습니다. 사용자가 로그인하면 백엔드에서 세션을 생성하고, 인증 정보는 JavaScript에서 직접 접근할 수 없도록 `HttpOnly Cookie`에 저장하는 방식으로 인증을 구현했습니다. 전체 서비스에는 HTTPS도 적용되어 있어 팀 내부에서는 인증 보안이 충분히 갖춰져 있다고 판단하고 있었습니다.

서비스 구조는 다음과 같습니다.

```
Frontend
https://app.example.com

Backend API
https://api.example.com
```

로그인 성공 시 서버에서는 다음과 같은 세션 쿠키를 발급합니다.

```
Set-Cookie: sessionId=abc123;
HttpOnly;
Secure;
SameSite=None;
```

프론트엔드에서는 인증이 필요한 요청에 Cookie가 포함되도록 다음과 같이 API를 호출하고 있습니다.

```tsx
fetch("https://api.example.com/api/me", {
  credentials: "include",
});
```

그런데 보안 점검 과정에서 다음과 같은 문제가 발견되었습니다.

- 사용자가 작성한 게시글이나 프로필 소개글 일부를 HTML 형태로 표현하기 위해 **`dangerouslySetInnerHTML`**을 사용하고 있는데, 입력값에 대한 별도의 Sanitization 처리가 없습니다.
- 공격자가 게시글에 악성 JavaScript가 실행될 수 있는 HTML을 삽입했고, 다른 사용자가 해당 게시글을 열자 브라우저에서 공격자의 JavaScript가 실제로 실행되었습니다.
- 세션 쿠키에는 `HttpOnly`가 설정되어 있기 때문에 공격 코드에서 `document.cookie`를 실행해도 `sessionId` 값 자체는 읽을 수 없었습니다.
- 그런데도 악성 JavaScript에서 서비스의 API를 호출하자 브라우저가 기존 로그인 세션 Cookie를 자동으로 포함했고, **사용자의 비밀번호 변경, 프로필 수정 등 인증이 필요한 API 요청이 정상적으로 처리되었습니다.**
- 즉, 세션 Cookie 자체가 탈취된 것은 아니지만 공격자가 **로그인한 사용자의 브라우저를 이용해 사용자의 권한으로 요청을 실행할 수 있는 XSS 문제가 발생한 상태입니다.**

예를 들어 사용자 입력이 다음과 같이 그대로 HTML에 삽입될 수 있습니다.

```tsx
<div
  dangerouslySetInnerHTML={{
    __html: post.content,
  }}
/>
```

공격자는 이를 이용해 다음과 비슷한 코드를 실행시킬 수 있습니다.

```html
<img
  src="invalid"
  onerror="
    fetch('https://api.example.com/api/profile', {
      method: 'DELETE',
      credentials: 'include'
    })
  "
/>
```

---

동시에 **CSRF와 관련된 문제도 발견되었습니다.**

- 사용자가 `example.com` 서비스에 로그인한 상태에서 공격자가 만든 외부 사이트인 `https://evil.com`에 접속했습니다.
- 공격 사이트에는 사용자가 인지하지 못하는 상태에서 `api.example.com`으로 상태 변경 요청을 보내는 `<form>`이 포함되어 있었습니다.
- 공격자는 사용자의 `sessionId` 값을 전혀 알지 못했지만, 세션 Cookie가 `SameSite=None`으로 설정되어 있어 **Cross-Site 요청에도 브라우저가 Cookie를 포함할 수 있는 상태**였습니다.
- 백엔드 API에서는 요청에 유효한 `sessionId`가 존재하는지만 확인하고, **CSRF Token이나 요청의 `Origin`을 별도로 검증하지 않고 있었습니다.**
- 결국 공격 사이트에서 발생한 요청임에도 서버에서는 정상적으로 로그인한 사용자의 요청으로 판단해 계정 정보 변경 API를 처리했습니다.

예를 들어 공격 사이트에서 다음과 같은 요청을 만들 수 있습니다.

```html
<form action="https://api.example.com/api/email" method="POST">
  <input type="hidden" name="email" value="attacker@example.com" />
</form>

<script>
  document.forms[0].submit();
</script>
```

사용자가 별도의 버튼을 누르지 않았는데도 브라우저가 요청을 전송하고, 조건이 맞으면 인증 Cookie까지 함께 전송될 수 있습니다.

---

추가로 팀에서는 다음과 같은 혼란도 있는 상태입니다.

- 모든 서비스가 **HTTPS**를 사용하고 있기 때문에 XSS나 CSRF까지 어느 정도 방어되는 것으로 생각하고 있습니다.
- HTTPS가 적용되어 있으므로 `Secure` Cookie를 사용하고 있지만, **`Secure`가 정확히 어떤 공격을 막아주는지 설명할 수 있는 사람이 많지 않습니다.**
- `HttpOnly`를 사용하면 XSS 자체가 방어되는 것으로 알고 있는 개발자도 있습니다.
- `SameSite=None`을 사용하는 이유와 `Lax`, `Strict`와의 차이를 명확하게 이해하지 못하고 있습니다.
- CORS에서 `https://app.example.com`만 허용하고 있기 때문에 외부 사이트에서 API를 호출할 수 없다고 생각하고 있지만, 실제로는 CSRF 요청 자체가 발생할 수 있다는 점을 이해하지 못하고 있습니다.
- HTTPS 설정 과정에서 흔히 **“SSL 인증서를 적용했다”**고 표현하지만, SSL과 TLS가 어떤 관계이고 실제 HTTPS 통신에서 TLS가 어떤 역할을 하는지 정확히 설명하지 못하고 있습니다.

현재 서버 설정은 대략 다음과 같습니다.

```
Set-Cookie: sessionId=abc123;
HttpOnly;
Secure;
SameSite=None;
```

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

하지만 다음과 같은 방어는 적용되어 있지 않습니다.

```
XSS
- 사용자 입력 Sanitization 없음
- CSP 미적용

CSRF
- CSRF Token 없음
- Origin 검증 없음
- SameSite=None 사용

인증
- HttpOnly / Secure만 적용
```

따라서 현재 팀이 해결해야 할 핵심 문제는 단순히 **“보안 옵션을 더 붙이는 것”**이 아니라,

> **HTTPS/TLS, Secure, HttpOnly, XSS 방어, SameSite, CSRF 방어, CORS가 각각 어느 단계에서 어떤 공격을 막는지 이해하고, 서로 다른 보안 계층을 조합해 인증 구조를 다시 설계하는 것**

입니다.

### [해결]

1. XSS 해결 — 사용자 입력에서 악성 JavaScript가 실행되지 않게 하기 (Escape + Sanitization + CSP)

2. CSRF 해결 — 외부 사이트에서 인증 요청을 만들어내지 못하게 하기 (SameSite + CSRF Token + Origin 검증)

3. 인증 Cookie 설정 강화 (HttpOnly + Secure + Cookie 범위/세션 관리)

4. HTTPS/TLS + CORS 설정도 강화 (HTTPS/TLS + HSTS + CORS Allowlist)

### [목표]

- **HTTPS가 보호하는 범위와 보호하지 못하는 범위를 설명할 수 있다.**
  - HTTP와 HTTPS의 차이
  - SSL/TLS의 역할
  - 네트워크 구간 암호화가 XSS/CSRF를 막아주지 못하는 이유
- **XSS와 CSRF의 공격 원리를 구분할 수 있다.**
  - XSS: 공격자의 JavaScript가 우리 서비스의 Origin에서 실행되는 문제
  - CSRF: 사용자의 브라우저가 인증정보를 자동으로 포함하는 특성을 악용하는 문제
- **Cookie 기반 인증이 브라우저에서 어떻게 동작하는지 이해한다.**
  - Cookie가 요청에 자동으로 포함되는 이유
  - `HttpOnly`, `Secure`, `SameSite`의 역할
  - `SameSite=None` 사용 시 주의점
- **HttpOnly Cookie의 한계를 설명할 수 있다.**
  - JavaScript가 쿠키 값을 읽지 못하더라도 XSS를 통해 인증된 요청은 실행할 수 있는 이유
- **보안 기능을 하나의 해결책이 아니라 여러 방어 계층으로 설계할 수 있다.**

```
Network
   ↓
HTTPS / TLS

Authentication
   ↓
Secure + HttpOnly Cookie

Cross-Site Request
   ↓
SameSite + CSRF Token + Origin 검증

Browser Script
   ↓
XSS 방어 + CSP
```
