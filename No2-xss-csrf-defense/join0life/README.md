# 상황에 대한 판단
## 상황
우리 팀은 **React 기반** 웹 서비스를 운영하고 있습니다. 사용자가 **로그인하면 백엔드에서 세션을 생성**하고, 인증 정보는 JavaScript에서 직접 접근할 수 없도록 **`HttpOnly Cookie`**에 저장하는 방식으로 인증을 구현했습니다. 전체 서비스에는 **HTTPS**도 적용되어 있어 팀 내부에서는 인증 보안이 충분히 갖춰져 있다고 판단하고 있었습니다.

```
Frontend
https://app.example.com

Backend API
https://api.example.com
```

```
Set-Cookie: sessionId=abc123;
HttpOnly;
Secure;
SameSite=None;
```

```tsx
fetch("https://api.example.com/api/me", {
  credentials: "include",
});
```

## 판단
### 1. React 기반: JSX의 `{}` 텍스트 출력은 자동으로 Escape되어 XSS를 막아줌
XSS란? 공격자가 웹 페이지에 악성 Javascript 코드를 끼워 넣고, 그 코드가 다른 사용자의 브라우저에서 실행되는 공격

React는 `<div>{post.content}</div>`처럼 값을 넣으면 `<`, `>` 같은 문자를 Escape해서 **HTML이 아니라 문자열로** 출력함. 그래서 `<img onerror=...>`를 넣어도 태그로 해석되지 않고 글자 그대로 보임.

단, 아래 경우는 React가 막아주지 **않음**
- `dangerouslySetInnerHTML` → 이름 그대로 Escape를 끄고 HTML을 그대로 삽입함 (이번 케이스의 원인)
- `<a href={userInput}>`에 `javascript:alert(1)` 같은 URL이 들어가는 경우
- `ref`로 DOM에 직접 `innerHTML`을 넣는 경우

### 2. Frontend와 Backend는 Origin은 다르지만, Site는 같음
도메인 주소가 "같다"고 보면 안 되고, **Origin**과 **Site**를 구분해야 함.

| 구분 | 기준 | `app.example.com` vs `api.example.com` |
| --- | --- | --- |
| Origin | scheme + host + port | **다름** (cross-origin) → CORS 대상 |
| Site | scheme + 등록 가능한 도메인(eTLD+1) | **같음** (`https://example.com`, same-site) |

```
https://app.example.com:443/api/me
└─┬─┘   └──────┬──────┘ └┬┘└──┬──┘
scheme        host      port 경로
```

```
app . example . com
 │       │       └─ eTLD (공개 접미사: 누구나 이 아래에 도메인을 등록할 수 있는 부분)
 │       └───────── +1 (eTLD 바로 앞 한 칸)
 └───────────────── 서브도메인 (Site 비교에서는 무시)
```

`SameSite` 쿠키 속성은 Origin이 아니라 **Site 기준**으로 동작함.
그래서 `app` → `api` 요청은 same-site 요청이라 `SameSite=Lax`나 `Strict`로 바꿔도 쿠키가 정상적으로 전송됨. 즉, **이 서비스 구조에서는 `SameSite=None`을 쓸 이유가 없음.** 보안을 위해 `Lax`(또는 `Strict`)로 바꾸는 게 좋음.

참고로 `SameSite`는 요청(API 호출) 자체를 막는 게 아니라, **그 요청에 쿠키를 붙일지 말지**를 정하는 속성임.

```
SameSite=Strict // same-site 요청에만 쿠키 전송. 외부 사이트에서 링크를 클릭해 들어오는 경우에도 쿠키를 보내지 않음
SameSite=Lax    // same-site 요청 + 외부 사이트에서 링크 클릭처럼 페이지 자체가 이동하는(top-level) GET 요청에만 쿠키 전송.
                // 외부 사이트에서 보내는 POST form, fetch, iframe, img 요청에는 쿠키를 보내지 않음. (SameSite를 생략하면 Chrome 등 최신 브라우저는 Lax로 취급)
SameSite=None   // cross-site 요청을 포함해 모든 요청에 쿠키 전송. 반드시 Secure와 함께 써야 함
```

### 3. Set-Cookie 방식
백엔드가 로그인 성공 시 `Set-Cookie` 응답 헤더로 쿠키를 내려주고, 이후 브라우저가 요청마다 자동으로 쿠키를 붙이는 방식.
이 케이스에서 쿠키에 담기는 값은 JWT 같은 토큰이 아니라 **세션 ID**임. (실제 사용자 정보는 서버의 세션 저장소에 있고, 쿠키에는 그 세션을 찾는 키만 있음)

#### 세션 ID vs 토큰
둘 다 "이 요청을 보낸 사람이 로그인한 사용자다"를 증명하는 값이지만, **사용자 정보가 어디에 있는지**가 다름.

**세션 ID: 정보는 서버에 있고, 쿠키에는 번호표만 있음**
```
쿠키: sessionId=abc123   ← 의미 없는 랜덤 문자열 (번호표)

서버 세션 저장소 (메모리, Redis 등)
abc123 → { userId: 42, role: "user", 로그인시각: ... }
```
- 요청이 오면 서버가 저장소에서 `abc123`을 찾아서 누구인지 확인함
- 서버가 상태를 들고 있어서 **stateful** 방식
  - 데이터가 커졌을 때 서버에 부담이 있을 확률 높음
- 저장소에서 `abc123`을 지우면 그 순간 로그인이 끊김 → **강제 로그아웃이 쉬움**

**토큰 (보통 JWT): 정보가 토큰 안에 들어 있음**
```
eyJhbGciOi... . eyJ1c2VySWQiOjQyLCJleHAiOjE3... . 서명
   헤더          내용 { userId: 42, exp: 만료시각 }    위조 방지용
```
- 서버는 저장소를 조회하지 않고 **서명만 검증**해서 누구인지 확인함
- 서버가 상태를 안 들고 있어서 **stateless** 방식
- 한번 발급하면 만료 시각까지 유효해서 **중간에 무효화하기가 어려움** → 보통 Access Token은 짧게(예: 15분) 주고 Refresh Token으로 재발급하는 구조를 씀

| 구분 | 세션 ID | 토큰 (JWT) |
| --- | --- | --- |
| 사용자 정보 위치 | 서버 저장소 | 토큰 안 |
| 서버 상태 | stateful | stateless |
| 검증 방법 | 저장소 조회 | 서명 검증 |
| 강제 무효화 | 쉬움 (저장소에서 삭제) | 어려움 (블랙리스트 등 별도 장치 필요) |

⚠️ **저장 위치와 방식은 별개임.** 세션 ID도, JWT도 쿠키에 담을 수 있고 `Authorization` 헤더로 보낼 수도 있음. "쿠키면 세션, 헤더면 토큰"이 아님. 그래서 어느 방식이든 **쿠키에 담으면** 이번 케이스와 똑같이 XSS 악용과 CSRF 문제가 생김.

이번 케이스는 `sessionId=abc123`이니 **세션 ID 방식** → 3번 해결책의 "로그아웃 시 서버 세션 삭제", "로그인 시 세션 ID 재발급"을 서버 저장소 조작만으로 바로 적용할 수 있음.

백엔드에서 인증 정보를 주는 방식 2가지
- 쿠키 (`Set-Cookie`) → `HttpOnly`를 걸면 JS에서 값을 읽을 수 없음. 대신 브라우저가 자동으로 붙여주기 때문에 CSRF를 신경 써야 함
- 응답 body에 실어다주는 방식 → 프론트가 직접 저장(localStorage, 메모리 등)하고 `Authorization` 헤더에 붙여서 보냄. 자동으로 붙지 않으니 CSRF에는 강하지만, localStorage에 저장하면 XSS 발생 시 **토큰 자체가 탈취**됨
  - Next.js BFF(Backend For Frontend) 패턴: 백엔드가 body로 준 토큰을 **Next.js 서버**가 받아서 보관하고, 브라우저에는 BFF가 다시 `HttpOnly` 쿠키를 내려줌. 즉 토큰이 브라우저 JS까지 오지 않게 하는 구조라서 body 방식과 궁합이 좋음. (단, 브라우저 ↔ BFF 구간은 다시 쿠키 방식이므로 CSRF 방어는 여전히 필요)

### 4. `credentials: 'include'`
**cross-origin 요청에도 쿠키를 보내고 받도록 허용**하는 옵션.

app.example.com ->
api.example.com

- `fetch`의 기본값은 `credentials: 'same-origin'` → 같은 Origin에만 쿠키를 보냄
- `app.example.com` → `api.example.com`은 **다른 Origin**이라 기본값으로는 쿠키가 안 붙음 → 로그인 세션이 전달되지 않음
- 그래서 쿠키 기반 인증에서 프론트와 API의 Origin이 다르면 `credentials: 'include'`가 **필요**함

이때 서버도 아래처럼 응답해야 브라우저가 응답을 JS에 넘겨줌
```
Access-Control-Allow-Origin: https://app.example.com   // credentials 사용 시 * 불가, 정확한 Origin 지정 필요
Access-Control-Allow-Credentials: true
```

⚠️ 이번 XSS 공격 코드도 `credentials: 'include'`를 쓰고 있음. XSS 코드는 `app.example.com` 안에서 실행되기 때문에 정상 프론트 코드와 똑같이 쿠키가 붙고, CORS도 통과함.


# 문제에 대한 판단
## 문제
### XSS와 관련한 문제
- 사용자가 작성한 게시글이나 프로필 소개글 일부를 HTML 형태로 표현하기 위해 **`dangerouslySetInnerHTML`**을 사용
- 입력값에 대한 별도의 Sanitization 처리가 없습니다.

Escape vs Sanitization
- Escape: `<`를 `&lt;`로 바꾸는 식으로 **모든 HTML을 문자열로** 만들어버림 (React의 기본 동작). HTML 서식을 살릴 수 없음
- Sanitization: HTML은 살리되 **위험한 부분만 제거**하는 처리. `<b>`, `<p>` 같은 허용된 태그는 남기고 `<script>`, `onerror`/`onclick` 같은 이벤트 핸들러 속성, `javascript:` URL 등은 제거함. 보통 DOMPurify 같은 라이브러리를 사용 (허용 목록 방식)

- 게시글에 악성 JavaScript가 실행될 수 있는 HTML을 삽입
- 인증이 필요한 API 요청이 정상적으로 처리
- 공격자가 로그인한 사용자의 브라우저를 이용해 사용자의 권한으로 요청을 실행할 수 있는 XSS 문제가 발생한 상태

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

### CSRF와 관련한 문제
CSRF란? cross-site request forgery로, 사용자가 로그인된 상태를 악용해서 공격자가 사용자의 의도와 상관없이 요청을 보내게 만드는 공격
- 세션 Cookie가 `SameSite=None`
- 백엔드 API에서 CSRF Token이나 요청의 `Origin`을 별도로 검증하지 않고 있었음

XSS vs CSRF
- XSS: 공격 코드가 **우리 사이트(`app.example.com`) 안에서** 실행됨 → 같은 Origin이라 SameSite, CORS, CSRF Token 모두 통과 가능 (페이지에 있는 CSRF Token도 읽을 수 있음)
- CSRF: 공격 코드는 **외부 사이트(`evil.com`)에서** 실행됨 → 응답을 읽을 수는 없고, 요청을 "보내기만" 함. 쿠키가 자동으로 붙는 것을 악용

### 추가 혼란
SOP란? same origin policy로, 브라우저가 보안상 어떤 웹 페이지가 자기와 같은 출처의 리소스만 자유롭게 읽을 수 있게 제한하는 정책.
- 핵심은 **"읽기"를 제한**하는 것이지, 다른 출처로 **요청을 "보내는 것" 자체를 막지는 않음** (form 전송, img 로드 등은 원래 가능)
- 그래서 SOP가 있어도 CSRF는 발생할 수 있음

-> 다른 출처의 리소스도 읽고 싶을 수 있음. 그래서 만든 것이 CORS.
CORS란? cross-origin-resource sharing으로, 브라우저가 다른 출처의 API를 호출할 때, 서버가 허용될 경우에만 응답을 프론트엔드 코드에서 읽을 수 있게 하는 정책. SOP를 **완화(예외 허용)**하기 위해 만든 것
* 브라우저가 적용하는 정책이라, 서버 to 서버 통신(curl, 백엔드 간 호출)에는 적용 X, 브라우저에서 보내는 요청에만 적용 O
* CORS는 **응답을 읽을 수 있는지**를 제어할 뿐, 요청이 서버에 도착하는 것을 항상 막아주지는 않음
  - `<form>` POST 같은 요청은 CORS 검사 대상이 아니라서, `evil.com`에서 보내도 서버에 그대로 도착하고 처리됨 → **CORS Allowlist로는 CSRF를 막을 수 없음**
  - 단, `Content-Type: application/json`이나 `DELETE` 같은 요청은 Preflight(OPTIONS)를 먼저 보내고, 서버가 허용하지 않으면 본 요청을 보내지 않음. 이건 부수적인 효과일 뿐, CSRF 방어책으로 기대하면 안 됨

## 판단
- `SameSite=Lax`로 바꾸면 `evil.com`에서 보내는 cross-site POST에는 쿠키가 붙지 않으므로 **CSRF를 크게 완화**할 수 있음. `app` → `api`는 same-site라 서비스 동작에는 영향 없음.
  - 단, Lax여도 top-level GET 이동에는 쿠키가 붙으므로 **GET 요청으로 상태를 변경하는 API가 있으면 안 됨**
  - 같은 Site의 다른 서브도메인(ex. `blog.example.com`)이 뚫리면 그곳에서 오는 요청은 same-site로 취급되어 쿠키가 붙음 → SameSite만으로는 부족하고 **CSRF Token, Origin 검증을 함께** 써야 함
  - SameSite는 **XSS는 막지 못함** (XSS 코드는 우리 사이트 안에서 실행되므로 same-site)
- `HttpOnly`는 **JS가 `document.cookie`로 쿠키 값을 읽지 못하게** 하는 속성임. XSS가 발생해도 세션 ID 자체를 **탈취해서 공격자 PC로 가져가는 것**은 막아줌.
  - 하지만 XSS 자체를 막는 건 아님. 쿠키 값을 몰라도 브라우저가 알아서 쿠키를 붙여주기 때문에, XSS 코드는 사용자 권한으로 API를 호출할 수 있음 → XSS 자체는 Sanitization, CSP로 막아야 함
- 쿠키가 요청에 자동으로 포함되는 건 `HttpOnly`의 특성이 아니라 **쿠키 자체의 특성**임. 이 자동 포함 때문에 쿠키 기반 인증은 CSRF 문제를 가질 수 있음.
공격자가 사용자의 브라우저로 요청을 보내면 쿠키가 같이 전송될 수 있기 때문이다. 이 경우는 SameSite 옵션, CSRF token, Origin/Referer 검증 같은 방어가 함께 필요하다.

# 해결
## 해결 방법
인증 Cookie 설정 강화 (HttpOnly + Secure + Cookie 범위/세션 관리)

## 이 방법은 어떤 방법인가
**인증 수단인 세션 쿠키 자체를 지키고, 뚫렸을 때의 피해 범위를 줄이는 방법**

- 1번(XSS 방어)과 2번(CSRF 방어)은 공격이 **일어나지 않게** 막는 계층
- 3번은 공격이 일어나더라도 세션이 **털리지 않게**, 털리더라도 **적게, 짧게** 피해를 입도록 하는 계층
- 그래서 3번만으로는 이번 케이스의 XSS/CSRF를 해결할 수 없고, 1번·2번과 함께 써야 함

## 3번이 다루는 네 가지

| 항목 | 막는 것 | 이번 케이스 상태 |
| --- | --- | --- |
| **HttpOnly** | XSS 코드가 세션 ID를 읽어서 공격자 서버로 보내는 것 (세션 탈취) | ✅ 적용됨 |
| **Secure** | HTTP 평문 구간에서 쿠키가 노출되는 것 (스니핑) | ✅ 적용됨 |
| **쿠키 범위** (Domain / Path / `__Host-` / SameSite) | 쿠키가 필요 없는 곳까지 전송되는 것, 다른 서브도메인이 쿠키를 덮어쓰는 것 | ❌ `SameSite=None`, 범위 제한 없음 |
| **세션 관리** (재발급 / 만료 / 무효화) | 세션 고정 공격, 탈취된 세션을 오래 쓰는 것 | ❌ 언급 없음 |

HttpOnly와 Secure는 이미 적용되어 있으므로, 이 방법의 핵심은 **쿠키 범위와 세션 관리를 추가하는 것**

## 이번 케이스에 적용하면

```
# Before
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=None

# After
Set-Cookie: __Host-sessionId=<랜덤값>; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=1800
```

### 1) 쿠키 범위 좁히기
- **`SameSite=None` → `Lax`**: `evil.com`의 form POST에 쿠키가 붙지 않으므로 이번 케이스의 CSRF를 직접 완화함 (2번 CSRF 방어와 겹치는 부분)
  - `app` → `api`는 same-site라 서비스 동작에는 영향 없음
- **`__Host-` 접두사 + Domain 생략**: 쿠키가 `api.example.com`에만 묶이고, 다른 서브도메인이 같은 이름의 쿠키로 덮어쓸 수 없음 (Cookie Tossing, 세션 고정 방지)
- **`Path=/`**: `__Host-` 접두사의 필수 조건. 인증이 전체 API에 필요하므로 `/`로 둠

### 2) 세션 관리
- **만료**: 쿠키 `Max-Age` + 서버 세션의 비활동 만료(idle)와 절대 만료(absolute)를 함께 둬서, 세션이 탈취·악용될 수 있는 시간을 줄임
- **로그인 시 세션 ID 재발급**: 로그인 전 세션 ID를 버리고 예측 불가능한 새 ID를 발급 → 세션 고정 공격 방지
- **로그아웃 시 서버 세션 삭제**: 쿠키만 지우는 게 아니라 서버의 세션 저장소에서도 삭제해야 함
- **민감한 작업은 재인증**: 비밀번호 변경, 이메일 변경 시 현재 비밀번호를 다시 받음
  - 이번 XSS 시나리오의 "비밀번호 변경 요청이 그대로 처리됨"을 직접 막을 수 있어서, 3번 중 가장 효과가 큰 조치

## 한계
- 브라우저가 쿠키를 자동으로 붙이는 한, **XSS 코드가 사용자 권한으로 요청하는 것 자체는 막지 못함** → 1번(Escape + Sanitization + CSP)이 필요
- SameSite=Lax는 같은 Site의 다른 서브도메인에서 오는 요청은 막지 못함 → 2번(CSRF Token + Origin 검증)이 필요

## 한 줄 요약
> HttpOnly/Secure는 **쿠키 값을 훔쳐가는 것**만 막는다. 쿠키가 **어디로(범위), 언제까지(수명)** 쓰일 수 있는지까지 좁혀야 인증 Cookie가 '강화'된다. 다만 브라우저가 쿠키를 자동으로 붙이는 한 XSS 코드가 사용자 권한으로 요청하는 것 자체는 막지 못하므로, 1번(XSS 방어)과 함께 가야 한다.

## 참고 자료
> 2026-09-28 기준으로 수집했고, 주제별로 최신 글이 위에 오도록 정리함. ⚠️ 표시는 읽을 때 주의할 부분.

### 쿠키 속성 전반 (HttpOnly / Secure / SameSite)
- [React SPA 인증 실전 가이드 — 쿠키 vs 스토리지, Axios 인터셉터, XSS/CSRF 방어 완전 정복](https://www.youngju.dev/blog/architecture/2026-03-08-sso-cookie-jwt-auth-react) (개인 테크 블로그, 2026-03-08)
  - `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=900` 조합 예시, Double Submit Cookie 방식의 CSRF Token, "HttpOnly여도 XSS가 나면 사용자 세션으로 API 호출 가능"이라는 한계까지 다룸. React 기준이라 이번 케이스와 가장 가까움
  - ⚠️ "Lax = GET 허용, POST 차단"이라고 단순화되어 있음. 정확히는 **외부 사이트에서의 top-level GET 이동**만 허용
- [강력한 세션 강화를 위한 HttpOnly, Secure, SameSite 속성 이해 및 쿠키 관리](https://leapcell.io/blog/ko/gangryeokhan-sesyeon-ganghwa-reul-wihan-http-only-secure-samesite-sokseong-ihae-mit-kuki-gwanli) (Leapcell 기술 블로그, 2025-09-30)
  - 세 속성의 역할 + express-session 설정 예제. 번역 글이라 문장이 조금 어색함
- [🍪 HttpOnly Secure 쿠키 - 왜 안전한가](https://velog.io/@thekim12/HttpOnly-Secure-%EC%BF%A0%ED%82%A4-%EC%99%9C-%EC%95%88%EC%A0%84%ED%95%9C%EA%B0%80) (velog, 2024-12-24)
  - Spring Boot 기준. 쿠키 방식의 한계(서버에서 강제 무효화하려면 Redis 블랙리스트 등 별도 장치 필요)도 언급
  - ⚠️ "HttpOnly가 XSS 공격으로부터 안전하다"는 표현은 부정확함. 막는 건 **쿠키 탈취**뿐임
- [HTTP Cookie 와 SameSite 정책에 대해서](https://velog.io/@rookieand/HTTP-Cookie-%EC%99%80-SameSite-%EC%A0%95%EC%B1%85%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C) (velog, 2023-05-11)
  - 퍼스트/서드파티 쿠키, SameSite 도입 배경 설명
  - ⚠️ Site(eTLD+1)와 Origin 구분이 없어서, 위 판단 섹션의 Origin vs Site 표와 같이 보는 게 좋음

### 세션 관리 (세션 재발급 / 만료 / 무효화)
- [세션 하이재킹 완벽 가이드 - 쿠키 탈취(XSS) 공격과 HttpOnly 방어법](https://www.yolog.co.kr/post/security-session-hijacking/) (개인 테크 블로그, 2025-09-03)
  - Node.js 실습으로 "HttpOnly 없을 때 탈취 → 붙였을 때 방어"를 직접 보여줌. 세션 ID는 `crypto.randomBytes()`처럼 예측 불가능하게 만들어야 한다는 내용 포함. 발표 데모 참고용으로 좋음
- [세션 하이재킹(Session Hijacking)이란 무엇이며, 어떻게 방어하나요?](https://velog.io/@ouk/%EC%84%B8%EC%85%98-%ED%95%98%EC%9D%B4%EC%9E%AC%ED%82%B9Session-Hijacking%EC%9D%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B4%EB%A9%B0-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%B0%A9%EC%96%B4%ED%95%98%EB%82%98%EC%9A%94) (velog, 2025-01-01)
  - 로그인 시 세션 ID 재발급, 비활동 타임아웃(예: 30분), 쿠키 속성, HTTPS를 방어 계층으로 정리
- [세션 고정 취약점](https://velog.io/@inmo/%EC%84%B8%EC%85%98-%EA%B3%A0%EC%A0%95-%EC%B7%A8%EC%95%BD%EC%A0%90) (velog, 2023-07-14)
  - 세션 고정 공격 원리: 공격자가 심어둔 세션 ID로 피해자가 로그인하게 만드는 공격 → **로그인할 때마다 새 세션 ID 발급 + 기존 ID 파기**로 방어
- [동시 세션 제어, 세션 고정 보호, 세션 정책](https://velog.io/@seungju0000/%EB%8F%99%EC%8B%9C-%EC%84%B8%EC%85%98-%EC%A0%9C%EC%96%B4-%EC%84%B8%EC%85%98-%EA%B3%A0%EC%A0%95-%EB%B3%B4%ED%98%B8-%EC%84%B8%EC%85%98-%EC%A0%95%EC%B1%85) (velog, 2022-04-06)
  - 오래된 글이지만 Spring Security의 `changeSessionId()`, 동시 로그인 제한(`maximumSessions`)을 실제 설정으로 보여줌. 백엔드에서는 어떻게 적용하는지 보여줄 때 참고

### 쿠키 범위 (Domain / Path / `__Host-` 접두사)
> 한국어 블로그 중에는 이 주제를 제대로 다룬 최신 글을 찾지 못해서 MDN 한국어 문서를 메인으로 봄.
- [HTTP 쿠키 - MDN](https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/Cookies) (공식 문서, 지속 갱신)
  - `Domain`을 생략하면 쿠키를 설정한 호스트에만 전송(서브도메인 제외), 지정하면 서브도메인까지 포함 → **인증 쿠키는 Domain을 생략하는 게 범위가 가장 좁음**
  - `Expires`/`Max-Age`가 없으면 브라우저 종료 시 삭제되는 세션 쿠키
- [Set-Cookie - MDN](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Set-Cookie) (공식 문서, 지속 갱신)
  - `__Host-` 접두사: `Secure` 필수 + `Path=/` 필수 + `Domain` 금지. 다른 서브도메인이 같은 이름의 쿠키를 덮어쓰는 공격(Cookie Tossing, 세션 고정)을 브라우저 차원에서 막아줌
  - 적용 예시: `Set-Cookie: __Host-sessionId=abc123; Path=/; HttpOnly; Secure; SameSite=Lax`

### 세션 타임아웃 기준 (영문 공식 자료)
> 한국어 블로그 중에는 기준을 근거와 함께 정리한 글이 없어서 OWASP 원문을 첨부함.
- [OWASP Session Timeout](https://owasp.org/www-community/Session_Timeout) / [OWASP WSTG - Testing Session Timeout](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/07-Testing_Session_Timeout)
  - 비활동 타임아웃(idle)과 절대 만료(absolute)를 둘 다 두고, 로그아웃과 만료 시에는 **서버의 세션을 무효화**해야 함 (쿠키만 지우는 것으로는 부족함)

### 참고용으로 제외한 자료
- [서로 다른 도메인에서 쿠키 사용하기 (React, Express)](https://velog.io/@jojeon4515/%EC%84%9C%EB%A1%9C-%EB%8B%A4%EB%A5%B8-%EB%8F%84%EB%A9%94%EC%9D%B8%EC%97%90%EC%84%9C-%EC%BF%A0%ED%82%A4-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-React-Express) 등
  - "app과 api 도메인이 다르면 `SameSite=None; Secure`로 바꾸면 된다"는 류의 글이 검색 상위에 많음. 이번 케이스처럼 **같은 Site의 서브도메인이면 None이 필요 없고**, 이 설정이 바로 CSRF 문제의 원인이라 참고하지 않는 게 좋음
