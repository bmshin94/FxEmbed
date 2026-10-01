# FxEmbed 전수조사 & 활용 전략 정리 (한국어)

> 작성일: 2026-10-01
> 분석 대상 저장소: <https://github.com/bmshin94/FxEmbed>
> 업스트림 원본: <https://github.com/FxEmbed/FxEmbed>
> 공식 문서: <https://docs.fxembed.com> / API: <https://docs.fxembed.com/api/introduction>
> 작업 브랜치: `claude/laughing-allen-ftun67`

---

## 목차

1. [프로젝트 정체 — 전수조사 결과](#1-프로젝트-정체--전수조사-결과)
2. [쉽게 이해하기 (비유 설명)](#2-쉽게-이해하기-비유-설명)
3. [핵심 Q&A 7가지](#3-핵심-qa-7가지)
4. [수익화 아이디어 10선](#4-수익화-아이디어-10선)
5. [참고 링크](#5-참고-링크)

---

## 1. 프로젝트 정체 — 전수조사 결과

### 1.1 한 줄 정의

**FxEmbed** = 디스코드 / 텔레그램 등에서 X(트위터), Bluesky, TikTok, Instagram 링크를
**영상 · 다중 이미지 · 투표 결과까지 제대로 보이게 고쳐주는 임베드 프록시 서버** +
토큰 없이 쓸 수 있는 **공개 JSON REST API**.

- FxTwitter / FixupX / FxBluesky 가 모두 **같은 하나의 Cloudflare Worker**에서 동작
- 라이선스: **MIT** (상업적 이용 가능, 저작권 고지 필요)
- 원작자: `dangered wolf`

### 1.2 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 런타임 | Cloudflare Workers (`workerd`) |
| 라우팅 | Hono 4.x (`@hono/zod-openapi`) |
| 검증/스펙 | Zod 4 + OpenAPI 3.0 자동 생성 |
| 다국어 | i18next + i18next-icu (**29개 언어**, 한국어 포함) |
| 테스트 | Vitest + `@cloudflare/vitest-pool-workers` (Miniflare), **62개 테스트 파일** |
| 빌드 | esbuild (`esbuild.config.mjs`) |
| 에러추적 | Sentry (`@hono/sentry`, toucan-js) |
| 문서 | Astro Starlight (`docs/`) |
| 컨테이너 | Docker (`node:24-bookworm-slim` + wrangler) |
| 모노레포 | npm workspaces (`packages/*`) |

### 1.3 폴더 구조 전수조사

| 경로 | 규모 | 역할 |
|---|---|---|
| `src/worker.ts` | - | 진입점. **Host 헤더로 realm 분배** |
| `src/realms/` | twitter, bluesky, tiktok, instagram, api, bluesky-api, atmosphere | 도메인(realm)별 라우팅 트리 |
| `src/embed/status.ts`, `activity.ts` | - | 임베드 HTML / Mastodon 호환 activity JSON 생성 |
| `src/render/` | photo, video, instantview | 사진 · 영상 · 텔레그램 Instant View 렌더링 |
| `src/helpers/` | 30개 파일 | mosaic, translate, translateAI, palette, snowcode, pbsProxy, giftranscode, transcode, socialproof 등 |
| `src/providers/` | twitter, bluesky, mastodon, instagram, threads, tiktok | 플랫폼별 호스트 어댑터 / 라우트 등록 |
| `packages/atmosphere/` | **166개 TS 파일** | 공통 프로토콜 레이어 + 통일 응답 봉투(unified envelope) |
| `packages/atmosphere/src/transports/` | 4종 | `public` / `anonymous-proxy` / `proxy-relay` / `authenticated` |
| `i18n/` | 29개 언어 디렉터리 | Crowdin 연동 번역 리소스 |
| `test/` | 62개 `.test.ts` + mock fixtures | Miniflare에서 실행, **실제 API 자격증명 불필요** |
| `docs/` | Astro | docs.fxembed.com 소스 + OpenAPI 스펙 추출 스크립트 |
| `tools/` | 4개 스크립트 | `credential-tools.mjs`(암호화/업로드), `add_csrf.mjs`, `stripcredentials.mjs`, `change_country.ts` |
| `.github/workflows/` | build, deploy, eslint, tests | CI/CD |
| `branding.example.json` | zones 배열 | **도메인별 이름 / 색상 / 파비콘 / 리다이렉트 커스터마이징** |
| `AGENTS.md` | 7.5KB | **AI 코딩 에이전트용 작업 지침서** (환경변수 추가 시 수정할 6개 위치 등) |
| `CLAUDE.md` | 1.4KB | 이 포크에서 추가한 Claude 페르소나 가이드 |

규모: `src` 91개 TS 파일 / 13,397줄, `packages/atmosphere` 166개 파일. 전체 약 42,600줄.

### 1.4 동작 원리 (핵심 아이디어)

```
요청 수신
  → Host 헤더로 realm 판별 (fxtwitter.com / fxbsky.app / api.fxtwitter.com / d.* ...)
  → User-Agent 검사
       ├─ 봇 (discordbot|telegrambot|facebook|whatsapp|revoltchat|iframely ...)
       │     → OpenGraph / twitter:card 메타태그 HTML 반환
       └─ 사람 브라우저
             → 원본 플랫폼으로 302 리다이렉트 (추적 파라미터 ?s= &t= 제거)
```

`src/worker.ts`의 `embeddingClientRegex`가 봇 판별을 담당한다.

### 1.5 "Realm" 개념

소스 주석 그대로: 도메인이 아니라 **realm**이라 부르는 이유는
`fxtwitter.com`과 `fixupx.com`은 내용이 동일하지만 `api.fxtwitter.com`은 다르기 때문.
워커 하나에 수십 개 도메인을 연결해도 올바른 콘텐츠로 라우팅된다.

### 1.6 URL 모디파이어 (서브도메인 플래그)

| 모디파이어 | 문법 | 효과 |
|---|---|---|
| Direct media | `d.` 접두사 또는 `.mp4`/`.jpg` 접미사 | 미디어 파일 직접 링크 |
| Select photo | `/photo/{n}` | 특정 이미지 선택 |
| Translate | `/{lang}` | 포스트 본문 번역 |
| Mosaic | `m.` | 여러 이미지를 한 장으로 합성 |
| Gallery | `g.` | 작성자 + 미디어만 표시 |
| Text-only | `t.` | 미디어 제외, 텍스트만 |
| Instant View | `i.` | 텔레그램 Instant View (스레드 전체 펼침) |
| Old embeds | `o.` | 구버전 디스코드 임베드 스타일 |

### 1.7 공개 API v2 엔드포인트

베이스: `https://api.fxtwitter.com` / `https://api.fxbsky.app` (+ `api.atmosphere.tools`)

```
GET /2/status/{id}                 GET /2/status/{id}/quotes
GET /2/status/{id}/reposts         GET /2/thread/{id}
GET /2/conversation/{id}           GET /2/profile/{handle}
GET /2/profile/{handle}/about      GET /2/profile/{handle}/statuses
GET /2/profile/{handle}/media      GET /2/profile/{handle}/articles
GET /2/profile/{handle}/followers  GET /2/profile/{handle}/following
GET /2/search                      GET /2/search/users
GET /2/trends                      GET /2/typeahead
GET /2/openapi.json                (OpenAPI 3.0 스펙)
```

- 응답에 항상 `code` 필드 포함 (HTTP 상태와 동일) → 에러 체크 용이
- 목록 엔드포인트는 `cursor.top` / `cursor.bottom` 기반 페이지네이션
- **레이트리밋: IP당 분당 1,000 요청** (초당 약 16.7)
- Atmosphere API는 `User-Agent` 헤더 필수 (없으면 401)

### 1.8 지원 플랫폼 현황

| 플랫폼 | 상태 | 비고 |
|---|---|---|
| X / Twitter | ✅ 가장 완성도 높음 | GraphQL + 게스트 토큰, 선택적 계정 프록시 |
| Bluesky | ✅ 완성 | AT Protocol, OAuth/DPoP 구현까지 포함 |
| TikTok | ⚠️ 부분 | 공개 SSR 페이지 파싱만. 서명(X-Gorgon/X-Ladon/X-Argus) 때문에 댓글·검색·트렌드 미지원 |
| Instagram | ⚠️ 부분 | 로그아웃 경로는 동작, 프록시 전용 라우트는 자격증명 없으면 501 |
| Threads | ⚠️ 부분 | Instagram 자격증명 풀 재사용 |
| Mastodon | ✅ | `src/providers/mastodon/` |

### 1.9 프라이버시 정책 (문서 기준)

- 수집: 런타임 에러(URL/기본 헤더), 계정 프록시 에러(포스트 ID), 익명 통계(Analytics Engine)
- 미수집: 링크를 올린 사람, 클릭한 사람, 사용자/포스트 식별 가능 데이터
- 추적 파라미터(`?s=`, `&t=`) 자동 제거

### 1.10 이 저장소가 주는 가치

1. **즉시 실용** — 설치 없이 링크에 `fx` 붙이기만 하면 됨
2. **개발 자산** — X 공식 API(Basic $200/월, Pro $5,000/월) 없이 무료로 소셜 데이터 JSON 확보
3. **학습 교재** — 엣지 런타임 + Hono + Zod OpenAPI + 모노레포 + Miniflare 테스트 + Astro 문서의 모범 사례
4. **상업적 활용** — MIT 라이선스이므로 포크 후 자체 브랜드 서비스 가능

---

## 2. 쉽게 이해하기 (비유 설명)

### 2.1 배달 대행 비유

- 디스코드(배달앱)가 트위터에 "영상 좀 보내줘" 요청 → 트위터는 썸네일 1장만 성의없이 줌
- FxEmbed는 중간에 끼어든 **친절한 대행 사장님**: 직접 영상 받아오고, 사진 여러 장 예쁘게 포장하고,
  투표 결과까지 적어서 배달앱이 알아듣는 **포장지(OpenGraph 메타태그)** 로 바꿔 전달

### 2.2 Realm = 건물 입구

```
fxtwitter.com      → 트위터 임베드 방
fxbsky.app         → 블루스카이 방
api.fxtwitter.com  → 개발자용 JSON 방
d.fxtwitter.com    → 미디어 파일 직통 방
```

같은 서버인데 **어느 문으로 들어왔는지**에 따라 다른 방으로 안내.

### 2.3 손님 구별 트릭 (가장 중요한 설계)

```
          요청
            │
      User-Agent 확인
       ┌────┴────┐
  "Discordbot"  사람 브라우저
       │            │
  임베드 HTML   원본으로 리다이렉트
  (사람은 못 봄)  (추적코드 제거)
```

봇만 메타태그를 읽고, 사람은 그냥 원본으로 이동 → **서버 한 대로 전 세계가 사용 가능**.

### 2.4 Atmosphere 패키지 = 만능 번역기

```
트위터 ─┐
Bluesky ─┼→ [@fxembed/atmosphere] → { code, status: { text, author, media } }
TikTok  ─┤                              (플랫폼 무관 동일 형태)
Insta   ─┘
```

플랫폼마다 데이터 형식이 다른데, 이걸 **하나의 공통 봉투**로 통일해준다.
→ 봇/에이전트를 만들 때 플랫폼별 코드를 따로 짜지 않아도 된다.

### 2.5 Cloudflare Worker = 전 세계 편의점

- 일반 서버: 한국에 1대 → 해외 사용자는 느림
- Worker: 전 세계 수백 개 도시에 동시 배치 → 어디서든 빠름, 요청 없으면 과금 없음

### 2.6 React SPA가 불가능한 이유 (쉬운 버전)

디스코드/텔레그램 봇은 **JavaScript를 실행하지 않는다.**
React SPA의 초기 HTML은 `<div id="root"></div>` 뿐이라 봇이 읽을 메타태그가 없다.
→ 이 문제는 **서버사이드 렌더링이 필수**다.

---

## 3. 핵심 Q&A 7가지

### Q1. 설치 및 사용법

**(A) 일반 사용자 — 설치 불필요**

```
x.com/user/status/123       → fixupx.com/user/status/123
twitter.com/user/status/123 → fxtwitter.com/user/status/123
bsky.app/profile/a/post/b   → fxbsky.app/profile/a/post/b
```

디스코드 팁: 트위터 링크를 보낸 뒤 `s/e/p` 입력 → `twittpr.com`으로 자동 수정.

**(B) API만 사용**

```bash
curl "https://api.fxtwitter.com/2/status/123456789"
curl "https://api.fxbsky.app/2/status/abc"
curl "https://api.fxtwitter.com/2/openapi.json"
# Atmosphere API는 User-Agent 필수
curl -H "User-Agent: MyBot/1.0 (+https://example.com)" "https://api.atmosphere.tools/2/openapi.json"
```

**(C) 자체 호스팅 (로컬 개발)**

```bash
git clone https://github.com/bmshin94/FxEmbed
cd FxEmbed
nvm use 24.14.1                      # Node 24 LTS (CI는 24.14.1)
npm install
cp .env.example .env
cp wrangler.example.toml wrangler.toml
cp branding.example.json branding.json

npm run build-local                  # Sentry 업로드 없이 빌드
npm run lint:eslint
npm run test                         # 62개 테스트 (자격증명 불필요)
npx wrangler dev --local             # http://localhost:8787
```

Host 헤더로 realm을 지정해야 테스트 가능:

```bash
curl -H "Host: fxtwitter.com" -H "User-Agent: Discordbot/2.0" \
  "http://localhost:8787/user/status/123"
```

**(D) Docker**

```bash
cp .env.example .env && cp wrangler.example.toml wrangler.toml && cp branding.example.json branding.json
docker compose up -d --build     # http://localhost:8787
docker compose down
```

**(E) 배포**

```bash
npm run deploy      # wrangler deploy --no-bundle
npm run tail        # 실시간 로그
wrangler secret put CREDENTIAL_KEY
```

### Q2. 플러그인? 스킬? MCP?

**정답: 전부 아니다. 독립 실행형 웹 서비스(Cloudflare Worker) + 공개 REST API.**

| 분류 | 해당 여부 | 설명 |
|---|---|---|
| 플러그인 | ❌ | 호스트 앱 확장이 아님 |
| Claude 스킬 | ❌ | `SKILL.md` 없음 (단 `AGENTS.md`/`CLAUDE.md`는 AI 에이전트용 지침서로 존재) |
| MCP 서버 | ❌ | MCP 프로토콜 구현 없음 |
| 웹 서비스 + REST API | ✅ | Hono 라우터 + OpenAPI 3.0 스펙 |

중요: **OpenAPI 스펙이 제공되므로 MCP 서버로 래핑하기가 매우 쉽다.**
→ 수익화 1순위 아이디어의 근거.

### Q3. API 토큰이 필요한가?

**공개 API 사용 시: 토큰 불필요. 가입/인증 없이 무료.** (IP당 분당 1,000 요청)

자체 호스팅 시 선택 항목:

| 용도 | 필요 항목 | 필수 여부 |
|---|---|---|
| 기본 임베드 | 없음 (게스트 토큰 자동) | 불필요 |
| 로그인 필수 콘텐츠 / 안정성 향상 | `credentials.json` (X `auth_token`+`ct0`, Bluesky 앱 패스워드, IG `sessionid`) | 선택 |
| 자격증명 암호화 | `CREDENTIAL_KEY` (wrangler secret) | 선택 |
| 에러 추적 | `SENTRY_DSN` | 선택 |
| 이미지 합성 | Mosaic 서버 별도 배포 | 선택 |
| 번역 | `POLYGLOT_ACCESS_TOKEN` | 선택 |

주의: 계정 쿠키 사용은 플랫폼 ToS 위반 소지 및 계정 정지 위험이 있다.
상업 서비스라면 `public` 트랜스포트(공개 데이터만) 사용을 권장.

### Q4. AI 에이전트 구축에 도움이 되는가 — 매우 그렇다

도움 되는 이유:

1. X 공식 API 비용(Basic $200/월, Pro $5,000/월) 회피 — 무료, 가입 없음
2. LLM 컨텍스트에 바로 넣을 수 있는 깔끔한 JSON (HTML 파싱 불필요)
3. 멀티플랫폼 통일 스키마 → 툴 하나로 여러 플랫폼 커버
4. OpenAPI 스펙 → Claude tool definition / LangChain tool / MCP 툴 자동 생성
5. `AGENTS.md`가 에이전트 친화적으로 작성되어 있어 AI 코딩 레퍼런스로도 유용

구현 가능한 에이전트 예시:

| 에이전트 | 사용 엔드포인트 |
|---|---|
| 소셜 모니터링 (브랜드 언급 → 슬랙 알림) | `/2/search` 폴링 |
| 트렌드 브리핑 (LLM 요약 → 메일) | `/2/trends` |
| 스레드 요약봇 | `/2/thread/{id}` |
| 미디어 아카이버 | `d.` 서브도메인 |
| 번역 리포스터 | `/{lang}` 모디파이어 |
| 디스코드 AI 큐레이터 | 임베드 + AI 코멘트 |

한계:

- **읽기 전용** — 포스팅/글쓰기 불가
- 비공개·보호 계정 접근 불가
- 공용 인스턴스 의존 → 상업용은 자체 호스팅 권장
- TikTok은 요청 서명 때문에 댓글/검색/트렌드 미지원

### Q5. 수익화 아이디어 → 4장 참조

### Q6. React나 PHP로 만들 수 있는가

| 스택 | 적합도 | 평가 |
|---|---|---|
| Cloudflare Workers + TS (현재) | ★★★★★ | 엣지 글로벌 배포, 콜드스타트 거의 없음, 무료 티어 넉넉 |
| PHP (Laravel / Slim) | ★★★ | 가능. 저렴한 호스팅 장점. 글로벌 지연·동시성은 약점 |
| **React SPA (CRA/Vite)** | ❌ | **구조적으로 불가능** |
| Next.js (SSR) | ★★★★ | React를 쓰려면 이 방식 (`generateMetadata()`) |
| Node + Hono/Express | ★★★★ | 현재 코드 상당 부분 재사용 가능 |

React SPA 불가 이유: 임베드 봇은 JS를 실행하지 않으므로 초기 HTML에 메타태그가 있어야 한다.
즉 **SSR이 요구사항**이다.

PHP 최소 프로토타입:

```php
<?php
$id = $_GET['id'] ?? '';
$ua = strtolower($_SERVER['HTTP_USER_AGENT'] ?? '');
$isBot = preg_match('/(discordbot|telegrambot|whatsapp|slack)/i', $ua);

if (!$isBot) {
    header("Location: https://x.com/i/status/$id", true, 302);
    exit;
}

$d = json_decode(file_get_contents("https://api.fxtwitter.com/2/status/$id"), true);
$s = $d['status'] ?? [];
?>
<!DOCTYPE html><html><head>
<meta property="og:title" content="<?= htmlspecialchars($s['author']['name'] ?? '') ?>">
<meta property="og:description" content="<?= htmlspecialchars($s['text'] ?? '') ?>">
<meta name="twitter:card" content="player">
<meta property="og:video" content="<?= htmlspecialchars($s['media']['videos'][0]['url'] ?? '') ?>">
</head><body></body></html>
```

실전 권장: 데이터 수집은 FxEmbed API에 위임하고, 렌더링/부가기능만 직접 구현하는 하이브리드.

### Q7. 유튜브 강의 영상 제작 가능성 — 매우 좋은 소재

장점: 결과가 시각적으로 즉시 확인됨, MIT 라이선스, 공감 가는 문제, 한국어 콘텐츠 희소.

10부작 커리큘럼 제안:

| 편 | 제목 | 길이 | 난이도 |
|---|---|---|---|
| 1 | 트위터 영상이 디스코드에 안 뜨는 이유 (`fx` 3글자의 마법) | 8분 | 입문 |
| 2 | OpenGraph 메타태그 완전정복 — 봇은 어떻게 링크를 읽나 | 12분 | 입문 |
| 3 | PHP 15줄로 임베드 서버 만들기 | 15분 | 입문 |
| 4 | Cloudflare Workers 입문 — 서버 없이 전 세계 배포 | 18분 | 중급 |
| 5 | Hono 라우팅 + Realm 패턴 설계 | 20분 | 중급 |
| 6 | User-Agent 분기 — 봇과 사람을 가르는 기술 | 14분 | 중급 |
| 7 | 실전: FxEmbed 자체 호스팅 A to Z | 25분 | 중급 |
| 8 | Zod + OpenAPI로 타입안전 API 만들기 | 22분 | 고급 |
| 9 | Miniflare로 Workers 테스트하기 (테스트 62개 분석) | 20분 | 고급 |
| 10 | FxEmbed API로 AI 에이전트 만들기 | 25분 | 고급 |

제작 주의사항:

- 계정 쿠키 우회를 "해킹 튜토리얼"처럼 다루면 수익창출 제한 위험 → "자체 호스팅 설정"으로 프레이밍
- 원작자(`dangered wolf`) 크레딧 + 저장소 링크 명시
- "Twitter/Tweet/X는 X Corp 상표이며 본 프로젝트는 무관" 디스클레이머 포함
- 실제 토큰/쿠키 화면 노출 금지 (`.env.example` 더미 값만 사용)

---

## 4. 수익화 아이디어 10선

평가 축: 수익성 / 난이도 / 합법성 / MVP 기간

### 1위. MCP 서버 SaaS — "Social Data MCP"

FxEmbed API를 MCP 서버로 래핑해 Claude / Cursor 사용자가 바로 연결해 쓰게 한다.
MCP 생태계가 급성장 중이지만 소셜 데이터 MCP는 희소 → 선점 기회.

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | $0 | 월 1,000 호출 |
| Pro | $19/월 | 월 5만 호출, 전 플랫폼 |
| Team | $99/월 | 월 50만 호출, 웹훅, 전용 키 |

수익성 ★★★★★ / 난이도 ★★ / 합법성 ✅ / MVP 1~2주

### 2위. 소셜 모니터링 SaaS (B2B)

브랜드·키워드 실시간 추적 + AI 감정분석 + 슬랙/디스코드 알림.
타겟: 마케팅 에이전시, 중소 브랜드, 엔터테인먼트 소속사.
가격: $49~$499/월 (기존 엔터프라이즈 솔루션 대비 압도적 저가).

수익성 ★★★★★ / 난이도 ★★★ / 합법성 ✅ / MVP 3~4주

### 3위. 디스코드 봇 프리미엄

임베드 + AI 요약 + 자동번역 + 아카이빙 봇.
수익: Discord 서버 서브스크립션($2.99~$9.99/월, 결제는 디스코드가 처리).

수익성 ★★★★ / 난이도 ★★ / 합법성 ✅ / MVP 2주

### 4위. 유튜브 + 온라인 강의 (리스크 최소)

```
유튜브 광고        월 30~100만원 (구독 1만 기준)
인프런/유데미 강의  ₩55,000 × 200명 = 1,100만원
멤버십/후원        월 20~50만원
1:1 코칭          시간당 5~10만원
```

수익성 ★★★ / 난이도 ★★ / 합법성 ✅ / 리스크 거의 없음

### 5위. 니치 임베드 서비스 (B2C)

한국 특화: 네이버 블로그·카페, 아프리카TV, 치지직, 트위치 클립 임베드 개선.
무료 + 기부 모델, 커스텀 도메인 유료($5/월).

수익성 ★★ / 난이도 ★★★ / 합법성 ⚠️ (각 플랫폼 ToS 확인 필요)

### 6위. 화이트라벨 셀프호스팅 대행

기업 자체 도메인(`embed.company.com`)으로 FxEmbed 설치·운영 대행.
`branding.json`의 zones 설정으로 커스텀 브랜딩이 기본 지원된다.
셋업비 100~300만원 + 월 운영 30~100만원.

수익성 ★★★★ / 난이도 ★★★ / 합법성 ✅

### 7위. 콘텐츠 아카이빙 SaaS

지정 계정의 포스트/영상 자동 백업. 타겟: 저널리스트, 연구자, 팬덤 아카이브, 법무팀.
$9~$99/월.

수익성 ★★★ / 난이도 ★★★ / 합법성 ⚠️ (저작권·개인정보 주의)

### 8위. AI 콘텐츠 리퍼포징 툴

트위터 스레드 → 블로그 글 / 뉴스레터 / 카드뉴스 자동 변환 (`/2/thread/{id}` + LLM).
크레딧 과금($0.1/변환) 또는 $29/월.

수익성 ★★★★ / 난이도 ★★ / 합법성 ⚠️ (원저작자 표기 필수)

### 9위. 오픈소스 스폰서십

GitHub Sponsors / Open Collective / Ko-fi. 직접 수익은 작지만 포트폴리오 신뢰도 향상.

수익성 ★ / 난이도 ★ / 합법성 ✅

### 10위. 기술 블로그 + 컨설팅 깔때기

"엣지 컴퓨팅 전문가" 포지셔닝 → 기업 컨설팅(프로젝트당 500~2,000만원).

수익성 ★★★★ / 난이도 ★★★★ (시간 소요 큼)

### 추천 로드맵

```
1~2개월   유튜브 3편 + 기술 블로그        → 리스크 0, 신뢰 확보
3~4개월   MCP 서버 무료 출시              → 트래픽/피드백 확보
5~6개월   MCP Pro 플랜 유료화             → 첫 MRR
7~12개월  모니터링 SaaS B2B 전환          → 본격 매출
```

### 수익화 전 체크리스트

| 항목 | 내용 |
|---|---|
| 라이선스 | MIT — 상업적 사용 가능, **저작권 고지 포함 필수** |
| 플랫폼 ToS | X의 ToS는 무단 스크래핑을 제한 → 상업화 전 법률 검토 권장 |
| 상표 | "FxTwitter", "X", "Twitter" 명칭 사용 금지, 독자 브랜드 사용 |
| 계정 쿠키 | 상업 서비스에 투입 시 계정 정지 리스크 → `public` 트랜스포트 권장 |
| 개인정보 | 국내 서비스 시 개인정보처리방침 필수 |
| 인프라 비용 | Workers 무료 10만 요청/일, 초과 시 100만 요청당 약 $0.30 |

가장 안전한 진입 순서: **유튜브/강의(리스크 0) → MCP 서버(난이도 낮고 수요 큼) → B2B SaaS**.
소셜 데이터 재판매는 법적 리스크가 있으므로 신중한 검토가 필요하다.

---

## 5. 참고 링크

| 항목 | URL |
|---|---|
| 이 저장소 (포크) | <https://github.com/bmshin94/FxEmbed> |
| 업스트림 원본 | <https://github.com/FxEmbed/FxEmbed> |
| 공식 문서 | <https://docs.fxembed.com> |
| API 레퍼런스 | <https://docs.fxembed.com/api/introduction> |
| 자체 호스팅 가이드 | <https://docs.fxembed.com/deployment> |
| FxTwitter OpenAPI 스펙 | <https://api.fxtwitter.com/2/openapi.json> |
| FxBluesky OpenAPI 스펙 | <https://api.fxbsky.app/2/openapi.json> |
| 서비스 상태 | <https://status.fxtwitter.com> |
| 번역 참여 (Crowdin) | <https://crowdin.com/project/fxtwitter> |
| Mosaic (이미지 합성) | <https://github.com/FxEmbed/mosaic> |
| 이슈 트래커 | <https://github.com/FxEmbed/FxEmbed/issues> |

---

*이 문서는 Claude Code 세션에서 저장소 전수조사를 바탕으로 작성되었습니다.*
