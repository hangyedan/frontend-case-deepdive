## A/B 테스트를 붙였더니 메인 페이지는 느려지고, 실험 결과는 믿을 수 없게 됐다

### [상황]

Next.js 기반 커머스 서비스입니다. 메인 페이지는 **정적 페이지(SSG/ISR)**로 만들어 CDN 캐시로 서빙하고 있습니다. 마케팅팀 요청으로 히어로 배너 문구와 CTA 버튼 색을 A/B 테스트하게 되었습니다.

**1차: 클라이언트에서 실험안 받아오기**

```tsx
"use client";

function HeroBanner() {
  const { variant, isLoading } = useExperiment("hero-banner-test"); // A/B 테스트 SDK

  if (isLoading) return <HeroSkeleton />;
  return variant === "B" ? <HeroB /> : <HeroA />;
}
```

- 실험 정보를 받을 때까지 스켈레톤이 보여서 **LCP가 약 1.2초 늘어남**
- 스켈레톤 대신 A안을 먼저 보여주면, B안 사용자에게 **A안 → B안 깜빡임** 발생
- B안 문구가 더 길어서 줄바꿈되고, 아래 상품 목록이 밀림 (**CLS 악화**)
- 첫 렌더링에서 쿠키를 읽어 바로 B안을 그리면 **Hydration mismatch** 발생
- 응답이 3초 넘게 걸리면 A안으로 대체하도록 처리 → **B안으로 기록됐지만 실제로는 A안을 본 사용자**가 생김
- 비로그인일 때는 기기 ID, 로그인 후에는 회원 ID로 배정 → **로그인 전후로 보이는 안이 바뀜**

**2차: 서버에서 쿠키를 읽어 배정하기**

- 깜빡임은 사라졌지만, 메인 페이지가 **요청마다 서버에서 렌더링(SSR)되도록 바뀌면서 CDN 캐시를 못 쓰게 됨** → TTFB 80ms → 400ms+

```
변경 전: Cache-Control: s-maxage=60, stale-while-revalidate
변경 후: Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
```

- 캐시를 살리려고 `public, s-maxage=600`으로 강제 변경 → **10분 동안 모든 사용자가 같은 안을 봄**

**실험 결과** *(예시 수치)*

| | A안 | B안 |
|---|---|---|
| 구매 전환율 | 2.8% | 3.1% |
| LCP (p75) | 3.4s | 2.9s |
| 사용자 수 | 48,120명 | 52,380명 |

- B안만 배너 이미지를 가벼운 파일로 바꿔 둠 → **전환율 차이가 디자인 때문인지 속도 때문인지 알 수 없음**
- 50:50으로 배정했는데 사용자 수가 **48:52**로 차이 남
- 전환율 0.3%p 차이가 **우연인지 확인하지 않고** B안 확정을 논의 중
- Lighthouse는 90점대인데, 실제 사용자 데이터(CrUX)는 LCP **"개선 필요"**

> **핵심 딜레마**: 깜빡임을 없애려면 서버에서 배정해야 하고, 서버에서 배정하면 캐시가 깨진다. 그리고 이 과정에서 측정한 숫자를 믿을 수 있는지도 모른다.

### [해결]

> - 아래 4가지는 서로 대체하는 방법이 아니라 **함께 적용해야 하는 계층**입니다. 하나만 고쳐서는 문제가 모두 해결되지 않습니다.
> - 각자 하나의 계층을 맡아 딥다이브하고, **그 계층에서 적용할 수 있는 구체적인 해결 방법**까지 조사해 주세요.
> - 권장 발표 순서: **2. 배정 → 3. 전달과 캐시 → 1. 렌더링 → 4. 측정**
>   (배정 기준이 있어야 서버/Edge 분기가 가능하고, 분기 방식이 정해져야 남은 렌더링 문제가 보이며, 측정은 구현을 모두 고친 뒤에도 항상 필요하기 때문입니다.)

**1. 렌더링: 실험이 첫 화면을 늦추거나 흔들지 않게 하기**
- CS 개념: Critical Rendering Path, LCP/CLS 측정 기준, Hydration
- 답해야 할 질문: 왜 스켈레톤은 LCP를 늦추고, A안을 먼저 그리면 깜빡이고, 쿠키를 바로 읽으면 Hydration mismatch가 나는가?

**2. 배정: 같은 사용자가 항상 같은 안을 보게 하기**
- CS 개념: 해시 함수와 결정적 버킷팅, 균등 분포, 사용자 식별자 설계
- 답해야 할 질문: 무작위 배정 대신 무엇으로 배정해야 하며, 로그인 전후나 기기가 바뀌어도 같은 안을 보게 하려면 어떻게 해야 하는가?

**3. 전달과 캐시: 캐시를 유지하면서 안별로 다른 화면 주기**
- CS 개념: 정적/동적 렌더링, HTTP 캐시와 CDN 캐시 키, `Vary`, Edge(Middleware) 분기
- 답해야 할 질문: 왜 쿠키를 읽으면 캐시가 깨지고, 강제로 캐시하면 모두 같은 안을 보며, 둘 다 피하려면 어디서 분기해야 하는가?

**4. 측정: 숫자를 믿어도 되는지 판단하기**
- CS 개념: Lab vs Field(RUM) 데이터, 백분위수(p75), 통계적 유의성, SRM
- 답해야 할 질문: Lighthouse와 CrUX는 왜 다르고, 0.3%p 차이와 48:52 비율은 믿을 수 있는 결과인가?

```
Browser      →  1. 렌더링  (CRP · LCP/CLS · Hydration)
Algorithm    →  2. 배정    (해시 · 버킷팅 · 식별자)
Network      →  3. 전달    (정적/동적 렌더링 · CDN 캐시 · Edge)
Data         →  4. 측정    (RUM · p75 · 유의성 · SRM)
```

### [참고] 실무에서 실제로 자주 일어나는 문제

1. **클라이언트 사이드 A/B 테스트의 깜빡임**: 서버가 그린 A안이 JS 로드 후 B안으로 바뀌는 현상 ([velog, 2025.10](https://velog.io/@crm03008/frontend-Next.js-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C%EC%9D%98-AB-%ED%85%8C%EC%8A%A4%ED%8A%B8))
2. **A/B 테스트용 콘텐츠 숨김으로 인한 LCP 지연**: 깜빡임을 막으려고 페이지를 숨기면 LCP 렌더링이 늦어짐 ([velog 번역글, 2024.10](https://velog.io/@superlipbalm/lcp-render-delay))
3. **쿠키를 읽으면 정적 페이지가 동적 렌더링으로 전환**: 서버에서 배정하면 Full Route Cache를 쓸 수 없게 됨 ([yceffort 블로그, 2025.12](https://yceffort.kr/2025/12/nextjs-caching-deep-dive))
4. **캐시가 다른 안을 잘못 서빙하는 문제**: A안으로 캐시된 페이지가 B안 사용자에게 전달됨 ([velog, 2025.10](https://velog.io/@crm03008/frontend-Next.js-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C%EC%9D%98-AB-%ED%85%8C%EC%8A%A4%ED%8A%B8))
5. **같은 사용자의 배정 일관성**: 실무 실험 플랫폼은 해시 기반으로 배정해 같은 사용자가 같은 안에 머물게 함 ([오늘의집 기술블로그, 2021.10](https://www.bucketplace.com/post/2021-10-29-%EC%98%A4%EB%8A%98%EC%9D%98%EC%A7%91-a-b-%EC%8B%A4%ED%97%98-%ED%94%8C%EB%9E%AB%ED%8F%BC-%EA%B5%AC%EC%B6%95%EA%B8%B0/))
6. **SRM(표본 비율 불일치)**: 50:50으로 나눴는데 실제 비율이 어긋나 실험 신뢰도가 떨어지는 현상 ([블로그, 2024.10](https://bongholee.com/a-bteseuteuyi-sinroeseongeul-jeohahaneun-hyeonsang-srm-sampling-ratio-mismatch/))
7. **Lighthouse와 실제 사용자 데이터(CrUX)의 차이**: 실험실 데이터와 필드 데이터가 다를 수 있는 이유 ([web.dev 한국어, 2022.07](https://web.dev/articles/lab-and-field-data-differences?hl=ko))
