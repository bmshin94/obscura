# Obscura 분석 및 활용 정리 노트 🚀

> Obscura 저장소를 처음 받아본 사람을 위한 한국어 분석 정리본입니다.
> 프로젝트가 무엇인지, 어떻게 설치하고 쓰는지, 어디에 활용하고 어떻게 수익화할 수 있는지를 한 문서에 담았습니다.

**작성일:** 2026-09-11

---

## 🔗 관련 링크

| 구분 | 주소 |
|------|------|
| **이 저장소 (fork)** | https://github.com/bmshin94/obscura |
| **원본 저장소 (upstream)** | https://github.com/h4ckf0r0day/obscura |
| 릴리스 (바이너리 다운로드) | https://github.com/h4ckf0r0day/obscura/releases |
| 공식 문서 | https://docs.obscura.sh |
| 공식 웹사이트 | https://obscura.sh |
| Docker Hub | https://hub.docker.com/r/h4ckf0r0day/obscura |
| 벤치마크 저장소 | https://github.com/h4ckf0r0day/obscura-benchmark |
| Hermes 에이전트 플러그인 | https://github.com/SGavrl/hermes-plugin-obscura |
| Cloudflare Kitesurf 엔지니어링 글 | https://blog.cloudflare.com/kitesurf/ |

**라이선스:** Apache-2.0 (상업적 사용 가능)

---

## 목차

1. [Obscura가 뭐야?](#1-obscura가-뭐야)
2. [폴더 구조 분석](#2-폴더-구조-분석)
3. [설치 및 사용법](#3-설치-및-사용법)
4. [플러그인? 스킬? MCP?](#4-플러그인-스킬-mcp)
5. [API 토큰이 필요할까?](#5-api-토큰이-필요할까)
6. [왜 깃허브에서 유명할까?](#6-왜-깃허브에서-유명할까)
7. [로컬 에이전트 구축에 도움될까?](#7-로컬-에이전트-구축에-도움될까)
8. [수익화 아이디어](#8-수익화-아이디어)
9. [React / PHP에서 쓸 수 있을까?](#9-react--php에서-쓸-수-있을까)
10. [시작 전 체크리스트](#10-시작-전-체크리스트)

---

## 1. Obscura가 뭐야?

**한 줄 요약: 크롬 없이 돌아가는, 초경량 + 스텔스 헤드리스 브라우저 엔진 (Rust 제작)**

컴퓨터가 사람 대신 웹사이트에 들어가서 내용을 읽고, 클릭하고, 화면을 캡처하는 프로그램입니다.
보통 이런 작업은 "헤드리스 크롬"으로 하는데, 크롬은 무겁고 봇 탐지에 바로 걸립니다.
Obscura는 **브라우저 자체를 Rust로 새로 만들어서** 이 두 문제를 동시에 해결했습니다.

### 성능 비교

| 항목 | Obscura | 헤드리스 크롬 |
|------|---------|--------------|
| 메모리 | **30 MB** | 200 MB+ |
| 바이너리 크기 | **70 MB** | 300 MB+ |
| 페이지 로드 | **85 ms** | ~500 ms |
| 시작 시간 | **즉시** | ~2초 |
| 봇 탐지 회피 | **내장** | 없음 |
| Puppeteer 호환 | ✅ | ✅ |
| Playwright 호환 | ✅ | ✅ |

### 페이지 로드 벤치마크

| 페이지 유형 | Obscura | Chrome |
|------------|---------|--------|
| 정적 HTML | **51 ms** | ~500 ms |
| JS + XHR + fetch | **84 ms** | ~800 ms |
| 동적 스크립트 | **78 ms** | ~700 ms |

### 스텔스(변장) 기능

- 세션마다 지문 랜덤화 (GPU, 화면, canvas, 오디오, 배터리)
- 실제와 유사한 `navigator.userAgentData` (Chrome 145 기준)
- `navigator.webdriver = undefined` (실제 크롬과 동일)
- 네이티브 함수 마스킹 (`Function.prototype.toString()` → `[native code]`)
- `event.isTrusted = true` 처리
- **트래커 도메인 3,520개 자동 차단**

---

## 2. 폴더 구조 분석

`crates/` 아래 9개 크레이트로 구성된 Rust 워크스페이스입니다. (총 약 15만 줄)

```
obscura-render   (72,203줄)  CSS 캐스케이드, 레이아웃, 텍스트 셰이핑, CPU 페인트
obscura-js       (29,653줄)  deno_core 기반 V8 런타임 + DOM 바인딩 (bootstrap.js + ops.rs)
obscura-cdp      (18,914줄)  Chrome DevTools Protocol 서버 (WebSocket, 디스패치, 도메인 핸들러)
obscura-browser  (10,777줄)  Page 타입, 네비게이션, 생명주기 이벤트
obscura-net       (5,676줄)  HTTP 클라이언트, 스텔스 클라이언트, 쿠키 저장소, robots 캐시, 트래커 차단
obscura-dom       (5,211줄)  DOM 트리 구현
obscura-cli       (3,834줄)  CLI 진입점 (fetch, serve, scrape, mcp)
obscura-mcp       (3,152줄)  Model Context Protocol 서버
obscura           (2,224줄)  임베더블 Rust 라이브러리 API (Browser, Page, Element, CookieStore)
```

### 요청 처리 흐름 (`Page.navigate` 기준)

```
CDP 클라이언트 (Puppeteer)
        │ WebSocket 프레임
        ▼
obscura-cdp/server.rs        accept, sessionId 라우팅
        ▼
obscura-cdp/dispatch.rs      메서드 라우터, v8_lock 획득
        ▼
obscura-cdp/domains/page.rs  Page.navigate 핸들러
        ▼
obscura-browser/page.rs      navigate_with_wait
        ├──► obscura-net/client.rs   HTTP 요청
        ├──► obscura-dom/tree.rs     HTML 파싱
        └──► obscura-js/runtime.rs   인라인 스크립트 실행
```

### 눈에 띄는 점

- **`obscura-render`가 압도적으로 큼(7만 줄)** — 크로미움을 쓰지 않고 CSS 레이아웃·폰트 래스터라이징·PDF 출력까지 직접 구현했습니다. 폰트 파일(Liberation, DejaVu, Noto Color Emoji)도 동봉되어 있습니다.
- **단일 V8 아이솔레이트** — 프로세스 내 모든 페이지가 하나의 V8 아이솔레이트를 공유하며, `v8_lock` 뮤텍스로 직렬화합니다.
- **`render-repros/` 폴더** — flex, grid, float, table, z-index 등 렌더링 검증용 HTML 픽스처가 80개 이상 들어 있고, 크로미움과 비교하는 파이썬 스크립트(`capture_chromium.py`, `paired_corpus.py`, `check.py`)도 포함되어 있습니다.
- **`tools/live-view.mjs`** — 헤드리스라 보이지 않는 화면을 로컬 브라우저로 실시간 중계해주는 도구 (의존성 없음, Node 21+ 필요).
- **테스트가 촘촘함** — iframe 이벤트 디스패치, 동시 네비게이션, 실행 컨텍스트 격리, 동시 아이솔레이트 정리 등 까다로운 케이스를 커버합니다.

---

## 3. 설치 및 사용법

### 3-1. 설치

#### 방법 A: 도커 (가장 간단)

```bash
docker run -d --name obscura -p 127.0.0.1:9222:9222 h4ckf0r0day/obscura
```

- 이미지는 `distroless/cc:nonroot` 기반, 약 57 MB (압축 기준)
- 셸도 패키지 매니저도 없고 uid 65532로 실행됨
- `--storage-dir`를 마운트하려면 해당 디렉터리가 uid 65532에 쓰기 가능해야 함
- `-p 9222:9222`는 모든 인터페이스에 노출되므로 위처럼 `127.0.0.1:`을 붙이는 것이 안전

#### 방법 B: 바이너리 다운로드 (Node·크롬 불필요)

```bash
# Linux x86_64
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-x86_64-linux.tar.gz
tar xzf obscura-x86_64-linux.tar.gz

# Linux ARM64
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-aarch64-linux.tar.gz

# macOS Apple Silicon
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-aarch64-macos.tar.gz

# macOS Intel
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-x86_64-macos.tar.gz

# Windows: releases 페이지에서 .zip 다운로드 후 압축 해제

# Arch Linux
yay -S obscura-browser

# NixOS
nix-env -iA nixpkgs.obscura
```

**릴리스 파일 접미사에 따라 기능이 다릅니다:**

| 접미사 | 렌더링 | 스텔스 전송 |
|--------|--------|------------|
| (없음) | ✅ | ❌ |
| `-stealth` | ✅ | ✅ ← **권장** |
| `-no-render` | ❌ | ❌ |
| `-no-render-stealth` | ❌ | ✅ |

> 릴리스 아카이브에는 `obscura`와 `obscura-worker`가 함께 들어 있습니다.
> 병렬 `scrape` 명령을 쓰려면 **두 파일을 같은 디렉터리에** 두어야 합니다.
> Linux 빌드는 Ubuntu 22.04 타깃(glibc 2.35+)입니다.

#### 방법 C: 소스 빌드

```bash
git clone https://github.com/h4ckf0r0day/obscura.git
cd obscura

# 렌더링만
cargo build --release -p obscura-cli --bins --features render

# 렌더링 + 스텔스
cargo build --release -p obscura-cli --bins --features render,stealth

# 렌더링 없이
cargo build --release -p obscura-cli --bins --no-default-features
```

- Rust 1.75+ 필요
- **첫 빌드는 약 5분 이상** (V8을 소스에서 컴파일, 이후 캐시됨)
- 스텔스 빌드는 BoringSSL도 컴파일하므로 추가 도구 필요:

```bash
sudo apt-get install build-essential cmake clang libclang-dev llvm-dev
```

### 3-2. 사용법 — 3가지 모드

#### 모드 1: `fetch` — 단일 페이지

```bash
obscura fetch https://example.com --dump text        # 텍스트
obscura fetch https://example.com --dump markdown    # 마크다운 (AI 입력용)
obscura fetch https://example.com --dump links       # 링크 목록
obscura fetch https://example.com --dump html        # 렌더링된 HTML
obscura fetch https://example.com --dump assets      # 서브리소스 URL 목록 (NDJSON)
obscura fetch https://example.com --eval "document.title"
obscura fetch https://example.com -s page.png        # 스크린샷
obscura fetch https://picsum.photos/200/300 --dump original > photo.jpg  # 원본 바이너리
```

| 플래그 | 기본값 | 설명 |
|--------|--------|------|
| `--dump` | `html` | `html`, `text`, `links`, `markdown`, `assets`, `original` |
| `--eval` | — | 평가할 JavaScript 표현식 |
| `--wait-until` | `load` | `load`, `domcontentloaded`, `networkidle0` |
| `--timeout` | `30` | 최대 네비게이션 시간(초) |
| `--wait` | 적응형(최대 5초) | 로드 후 안정화 대기 |
| `--selector` | — | 특정 CSS 셀렉터 대기 |
| `-s`, `--screenshot` | — | PNG 저장 (렌더 빌드 필요) |
| `--stealth` | off | 안티 디텍션 모드 |
| `--output` | — | 결과를 파일로 저장 |
| `--proxy` | — | HTTP/SOCKS5 프록시 |
| `--quiet` | off | 배너 숨김 |

#### 모드 2: `scrape` — 병렬 스크래핑

```bash
obscura scrape url1 url2 url3 \
  --concurrency 25 \
  --eval "document.querySelector('h1').textContent" \
  --format json

obscura --proxy http://127.0.0.1:8080 scrape https://a.com https://b.com
```

| 플래그 | 기본값 | 설명 |
|--------|--------|------|
| `--concurrency` | `10` | 병렬 워커 수 |
| `--format` | `json` | `json` 또는 `text` |
| `--quiet` | off | stderr 진행 표시 숨김 |

#### 모드 3: `serve` — CDP 서버

```bash
obscura serve --port 9222 --stealth --workers 4
```

| 플래그 | 기본값 | 설명 |
|--------|--------|------|
| `--port` | `9222` | WebSocket 포트 |
| `--proxy` | — | HTTP/SOCKS5 프록시 |
| `--stealth` | off | 안티 디텍션 + 트래커 차단 |
| `--workers` | `1` | 병렬 워커 프로세스 수 |
| `--obey-robots` | off | robots.txt 준수 |

기존 Puppeteer / Playwright 코드를 **거의 그대로** 붙일 수 있습니다.

```javascript
// Puppeteer (npm install puppeteer-core)
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserWSEndpoint: 'ws://127.0.0.1:9222/devtools/browser',
});
const page = await browser.newPage();
await page.goto('https://news.ycombinator.com');
const stories = await page.evaluate(() =>
  Array.from(document.querySelectorAll('.titleline > a'))
    .map(a => ({ title: a.textContent, url: a.href }))
);
await browser.disconnect();
```

```javascript
// Playwright (npm install playwright-core)
import { chromium } from 'playwright-core';

const browser = await chromium.connectOverCDP({ endpointURL: 'ws://127.0.0.1:9222' });
const page = await browser.newContext().then(ctx => ctx.newPage());
await page.goto('https://en.wikipedia.org/wiki/Web_scraping');
console.log(await page.title());
await browser.close();
```

### 3-3. 실시간 화면 보기

```bash
obscura serve --port 9222
node tools/live-view.mjs 9222 8080   # → http://localhost:8080
```

에이전트가 웹을 돌아다니는 모습을 실시간으로 볼 수 있어 디버깅에 매우 유용합니다.
페이지가 변할 때는 250ms, 유휴 상태에서는 1.5초 간격으로 적응형 캡처합니다.

### 3-4. 로컬 개발 서버 테스트

Obscura는 기본적으로 사설/내부 IP 접근을 차단합니다 (SSRF 방어).

```bash
obscura fetch http://127.0.0.1:3000 --allow-private-network --dump text
obscura serve --port 9222 --allow-private-network
# 또는 OBSCURA_ALLOW_PRIVATE_NETWORK=1
```

### 3-5. 주요 환경변수

| 변수 | 용도 |
|------|------|
| `OBSCURA_STEALTH` | 스텔스 모드 |
| `OBSCURA_PROXY` | 프록시 URL |
| `OBSCURA_ALLOW_PRIVATE_NETWORK` | 사설망 접근 허용 |
| `OBSCURA_OBEY_ROBOTS` | robots.txt 준수 |
| `OBSCURA_SCRIPT_DEADLINE_MS` | 스크립트 실행 예산 (기본 30초) |
| `OBSCURA_MODULE_BUDGET_MS` | 모듈당 예산 (기본 3초) |
| `OBSCURA_NAV_TIMEOUT_MS` | 네비게이션 타임아웃 |
| `OBSCURA_FETCH_TIMEOUT_MS` | fetch 타임아웃 |
| `OBSCURA_TIMEZONE` / `OBSCURA_GEOLOCATION` | 타임존 / 위치 위장 |
| `OBSCURA_PROFILE` / `OBSCURA_ROTATE_PROFILE` | 지문 프로필 / 회전 |
| `OBSCURA_SHOT_W` / `OBSCURA_SHOT_H` / `OBSCURA_SHOT_SCROLL_Y` | 캡처 크기·스크롤 |
| `OBSCURA_MCP_ALLOWED_ORIGINS` | MCP HTTP 오리진 허용목록 |
| `OBSCURA_NETWORK_BODY_BUFFER_BYTES` | 응답 본문 캐시 한도 (기본 2 MiB) |

무거운 SPA에서 V8 힙이 부족할 때:

```bash
obscura --v8-flags "--max-old-space-size=4096" fetch <url>
OBSCURA_SCRIPT_DEADLINE_MS=60000 obscura serve --port 9222
```

---

## 4. 플러그인? 스킬? MCP?

**정답: 독립 실행 프로그램이며, 그 안에 MCP 서버와 Claude 스킬 문서를 함께 품고 있습니다.**

```
Obscura (독립 Rust 바이너리)
 ├─ obscura serve  → CDP 서버 (Puppeteer / Playwright용)
 ├─ obscura mcp    → MCP 서버 (Claude 등 AI 클라이언트용)
 └─ skills/obscura/SKILL.md → Claude Skill 문서
```

| 질문 | 답 | 근거 |
|------|-----|------|
| 플러그인인가? | ❌ 아님 | Claude 플러그인 구조(`.claude-plugin/`)가 없음. 단 외부에 Hermes용 플러그인은 별도 존재 |
| 스킬인가? | ⭕ 일부 포함 | `skills/obscura/SKILL.md`가 실제로 존재 |
| MCP인가? | ⭕⭕ 핵심 기능 | `obscura mcp`로 MCP 서버 구동, 도구 40개 이상 |

### MCP 연결 방법

**Claude Code:**
```bash
claude mcp add obscura /path/to/obscura mcp
```

**Claude Desktop** (`claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "obscura": {
      "command": "/path/to/obscura",
      "args": ["mcp", "--stealth"]
    }
  }
}
```

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

**HTTP 전송:**
```bash
obscura mcp --http --port 3000                       # 기본 127.0.0.1 바인딩
obscura mcp --http --host 0.0.0.0 --port 3000        # 외부 노출
```

### 제공되는 MCP 도구

| 분류 | 도구 |
|------|------|
| 이동 | `browser_navigate`, `browser_back`, `browser_forward`, `browser_reload`, `browser_close` |
| 읽기 | `browser_snapshot`, `browser_markdown`, `browser_links`, `browser_extract`, `browser_search`, `browser_count`, `browser_get_attribute` |
| 조작 | `browser_click`, `browser_fill`, `browser_fill_form`, `browser_type`, `browser_press_key`, `browser_select_option`, `browser_scroll` |
| 폼 탐지 | `browser_interactive_elements`, `browser_detect_forms` |
| 대기/실행 | `browser_wait_for`, `browser_wait_for_text`, `browser_evaluate` |
| 시각 출력 | `browser_screenshot`, `browser_pdf` |
| 쿠키/저장소 | `browser_get_cookies`, `browser_set_cookie`, `browser_clear_cookies`, `browser_storage_state`, `browser_set_storage_state` |
| 탭 | `browser_tab_new`, `browser_tab_list`, `browser_tab_switch`, `browser_tab_close` |
| 진단 | `browser_network_requests`, `browser_console_messages` |

> ⚠️ 요소 참조(reference)는 네비게이션·클릭·스크롤·프레임워크 리렌더 후 무효화될 수 있습니다.
> 행동 전에 `browser_snapshot`을 새로 받아야 합니다.

### CDP 지원 도메인

| 도메인 | 메서드 |
|--------|--------|
| Target | createTarget, closeTarget, attachToTarget, createBrowserContext, disposeBrowserContext |
| Page | navigate, getFrameTree, lifecycleEvents, captureScreenshot, start/stopScreencast, printToPDF |
| Runtime | evaluate, callFunctionOn, getProperties, addBinding |
| DOM | getDocument, querySelector, querySelectorAll, getOuterHTML, resolveNode |
| Network | enable, setCookies, getCookies, setExtraHTTPHeaders, setUserAgentOverride |
| Fetch | enable, continueRequest, fulfillRequest, failRequest, takeResponseBodyAsStream |
| IO | read, close |
| Storage | getCookies, setCookies, deleteCookies |
| Input | dispatchMouseEvent, dispatchKeyEvent |
| LP | getMarkdown |

---

## 5. API 토큰이 필요할까?

**필요 없습니다. 완전 무료이며 인증 토큰이 전혀 없습니다.**

코드 전체의 환경변수를 확인한 결과, `API_KEY` / `TOKEN` / `LICENSE` 류의 인증 변수가 **하나도 없습니다.**
모든 환경변수는 동작 설정용입니다.

이유:
- **Apache-2.0** 오픈소스 (상업적 사용 가능)
- README 명시: *"오픈소스 엔진은 Apache-2.0 그대로, 완전 기능. 기능 게이팅 절대 없음"*
- 내 컴퓨터의 CPU로 도는 프로그램이라 외부 서버 호출이 없음

| 항목 | 비용 |
|------|------|
| Obscura 자체 | 무료 |
| 프록시 (IP 우회) | 선택 사항, 유료 |
| Obscura Cloud (호스팅판) | 준비 중 (대기명단) |
| Claude API 등 LLM | 별도 |

### ⚠️ 보안 주의: MCP HTTP 전송

HTTP 전송에는 **내장 인증이 없습니다.** 포트에 접근 가능한 사람은 누구나 브라우저를 조종할 수 있습니다.
제공되는 두 가지 보호 장치:

- **오리진 허용목록** — `OBSCURA_MCP_ALLOWED_ORIGINS`에 지정된 Origin만 허용, 그 외 403
- **바디 상한** — 요청 본문 16 MiB 제한

```bash
OBSCURA_MCP_ALLOWED_ORIGINS="https://app.example.com" obscura mcp --http --host 0.0.0.0
```

루프백 밖으로 노출할 때는 반드시 허용목록 + 리버스 프록시/네트워크 격리로 인증을 강제하세요.

---

## 6. 왜 깃허브에서 유명할까?

1. **타이밍** — AI 에이전트 붐. 에이전트마다 브라우저가 필요한데 크롬은 1개당 200MB. 에이전트 100개면 20GB. Obscura는 3GB.
2. **"크로미움 없음"의 임팩트** — 대부분 크롬 껍데기를 빌려 쓰는데, Obscura는 CSS 레이아웃(7만 줄), 폰트 래스터라이징, PDF 출력을 직접 구현했습니다.
3. **Cloudflare 레퍼런스** — Cloudflare가 새 에이전트 브라우저 **Kitesurf**의 첫 프로토타입을 Obscura를 Workers로 포팅하면서 시작했다고 README와 Cloudflare 블로그에 명시되어 있습니다.
4. **자극적인 수치 + 검증 가능성** — "30MB vs 200MB", "85ms vs 500ms" 같은 표는 SNS 확산에 유리하고, 벤치마크를 별도 저장소에 공개해 신뢰도를 확보했습니다.
5. **진입장벽 0** — 기존 Puppeteer/Playwright 코드를 그대로 사용 가능.

### 활발함 지표

커밋 로그를 보면 이슈 번호가 **949번대**까지 진행됐고, 외부 기여자 PR이 꾸준히 머지되고 있습니다.

```
#949 fix/890-render-transport      h4ckf0r0day
#948 fix/873-context-memory        h4ckf0r0day
#947 fix/945-cdp-worker-host       h4ckf0r0day
#931 fix/slot-element              michabbb      ← 외부 기여자
#914 fix/response-headers          mnaza         ← 외부 기여자
#887 fix/domparser-parsererror     ntdatt812     ← 외부 기여자
```

여기에 Trendshift 트렌딩 배지, 공식 사이트·문서 사이트·트위터 계정, 스폰서 다수까지 — 운영이 상당히 전문적입니다.

---

## 7. 로컬 에이전트 구축에 도움될까?

**매우 도움됩니다. 사실상 Obscura의 킬러 유즈케이스입니다.**

| 로컬 에이전트의 난제 | 기존 방식 | Obscura |
|---------------------|----------|---------|
| 무거움 | 2GB+ | 300MB |
| 봇 차단 | 바로 걸림 | 스텔스 내장 |
| 연동 번거로움 | 직접 개발 | MCP 한 줄 |

### 권장 구성

```
[ Claude / 로컬 LLM ]  ← 두뇌
        │ MCP
        ▼
[ obscura mcp --stealth ]  ← 손발
        │
        ▼
     [ 인터넷 ]
        │
[ live-view.mjs ]  ← 실시간 관찰 창
```

```bash
obscura serve --port 9222 --stealth --workers 4
claude mcp add obscura /path/to/obscura mcp
node tools/live-view.mjs 9222 8080
```

### 에이전트에 특히 유용한 기능

- **`--dump markdown` / `browser_markdown`** — HTML 노이즈 없는 마크다운 → **LLM 토큰 비용 절감**
- **`browser_snapshot`** — 페이지 요약 + 클릭 가능한 요소 목록을 AI가 먹기 좋게 정리
- **`browser_storage_state`** — 로그인 세션 저장/복원으로 반복 로그인 제거
- **`--concurrency`** — 다중 에이전트 동시 운영
- **`OBSCURA_ROTATE_PROFILE`** — 요청마다 지문 회전

### 솔직한 한계

- 영상 재생, WebGL, 서비스 워커, GPU/컴포지터 효과는 크로미움과 다르거나 미지원
- 롱테일 CSS와 플랫폼 폰트 래스터라이징 차이 존재
- → **텍스트를 읽고 클릭·입력하는 에이전트에는 최적**, **영상 소비형 에이전트에는 부적합**

---

## 8. 수익화 아이디어

### 8-0. 먼저: 우리의 무기는 "원가 구조"

#### 서버 원가 추정 (4vCPU / 8GB VPS, 월 약 2만원 기준)

| | 헤드리스 크롬 | Obscura |
|---|---|---|
| 인스턴스당 메모리 | 200~250MB | 30MB |
| 8GB 이론상 | 32개 | 266개 |
| CPU 감안 실제 동시 | 6~8개 | 30~40개 |
| 월 처리량 (보수적 추정) | 약 15만 페이지 | 약 130만 페이지 |

> ⚠️ 이 수치는 공개 스펙 기반 **추정치**입니다. 실제 사이트는 네트워크 대기 때문에 더 느릴 수 있으니
> 사업 계획 전에 반드시 본인 대상 사이트로 실측하세요. 다만 한 자릿수 배수의 우위는 분명합니다.

#### 진짜 원가는 서버가 아니라 프록시입니다

| 항목 | 1,000 페이지당 대략 원가 |
|------|----------------------|
| 서버 | 약 15원 |
| 데이터센터 프록시 | 약 50~200원 |
| 주거용(residential) 프록시 | 약 200~1,500원 |

→ **이미지·폰트·광고 차단으로 트래픽을 줄이는 것이 곧 마진**입니다.
Obscura는 `--stealth` 시 트래커 3,520개를 자동 차단하므로 여기서도 유리합니다.

```bash
# 트래픽 최소화 = 마진 극대화
obscura fetch $URL --stealth --dump markdown --wait-until domcontentloaded
```

---

### ① AI용 웹 리더 API — 난이도 ⭐ / 첫 매출 2~4주

**URL을 넣으면 깔끔한 마크다운을 반환하는 API.**

```
GET /read?url=https://news.site/article
→ { "title": "...", "markdown": "# 제목\n본문...", "links": [...] }
```

**타겟:** RAG/챗봇 개발자, n8n·Make 자동화 유저, 리서치 에이전트 스타트업

| 플랜 | 월 요금 | 페이지 수 | 페이지당 |
|------|--------|----------|---------|
| Free | 0원 | 500 | — |
| Hobby | 9,900원 | 20,000 | 0.5원 |
| Pro | 39,000원 | 150,000 | 0.26원 |
| Scale | 129,000원 | 700,000 | 0.18원 |

**원가:** Pro 15만 페이지 → 서버 약 2,300원 + 프록시 약 15,000원 ≈ **1.7만원 (마진 약 55%)**
프록시 없이 일반 사이트만 다루면 마진 90%대.

```javascript
// server.js — npm i express
import express from 'express';
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
const run = promisify(execFile);

const app = express();
const cache = new Map(); // 추후 Redis로 교체

app.get('/read', async (req, res) => {
  const { url, format = 'markdown' } = req.query;
  if (!url) return res.status(400).json({ error: 'url required' });

  // 캐시가 곧 마진: 같은 URL 재요청은 원가 0
  const key = `${url}:${format}`;
  if (cache.has(key)) return res.json({ ...cache.get(key), cached: true });

  try {
    const { stdout } = await run('obscura', [
      'fetch', url,
      '--dump', format,
      '--stealth',
      '--timeout', '20',
      '--quiet',
    ], { maxBuffer: 20 * 1024 * 1024, timeout: 30000 });

    const result = { url, content: stdout, chars: stdout.length };
    cache.set(key, result);
    setTimeout(() => cache.delete(key), 1000 * 60 * 30);
    res.json(result);
  } catch (e) {
    res.status(502).json({ error: 'fetch failed', detail: e.message });
  }
});

app.listen(3000);
```

**차별점:** 경쟁사(Firecrawl, Jina Reader, ScrapingBee 등)는 크롬 기반이라 원가가 높습니다.
절반 가격에 팔아도 마진이 남습니다. 여기에 **한국 리전·한국어 문서·원화 결제·카카오 지원**,
그리고 네이버 블로그/카페·티스토리·브런치 같은 **한국 사이트 특화 파서**를 붙이면 해외 서비스가 흉내내기 어렵습니다.

**리스크:** 경쟁 치열, 무료 남용 방지를 위한 API 키 + 레이트리밋 필수.

---

### ② 경쟁사 가격 모니터링 SaaS — 난이도 ⭐⭐⭐ / 첫 매출 6~10주

쿠팡·네이버쇼핑·11번가 셀러 대상으로 경쟁사 가격을 추적하고 알림을 보내는 서비스.

**타겟:** 스마트스토어·쿠팡 셀러(국내 수십만 명), 브랜드사 최저가 정책 관리팀

| 플랜 | 월 요금 | 추적 상품 | 주기 |
|------|--------|----------|------|
| Starter | 29,000원 | 50개 | 6시간 |
| Growth | 79,000원 | 300개 | 1시간 |
| Pro | 199,000원 | 1,500개 | 30분 |

**원가:** Growth 300개 × 24회 × 30일 = 21.6만 페이지/월 → 약 2.5만원 **(마진 약 68%)**

```bash
obscura scrape $(cat urls.txt) \
  --concurrency 25 \
  --stealth \
  --eval "document.querySelector('[class*=price]')?.textContent?.trim()" \
  --format json --quiet > prices.json
```

**차별점:** 기존 서비스(Prisync, Competera 등)는 해외 사이트 중심에 고가입니다.
한국 쇼핑몰 특화 + 절반 가격 + 낮은 원가 덕분에 "1시간 주기"를 기본 제공할 수 있습니다.

**리스크:**
- 대형 쇼핑몰은 봇 차단이 강해 스텔스 + 주거용 프록시가 필요 → 원가 상승
- 사이트 개편 시 셀렉터가 깨짐 → 지속적인 유지보수 인력 필요
- 약관 확인 필수. 공개 가격 정보 조회는 대체로 회색지대지만 과도한 요청은 문제가 됩니다

---

### ③ 스크린샷 / PDF API — 난이도 ⭐⭐ / 첫 매출 3~5주

```
GET /shot?url=...&width=1200&height=630&format=png
GET /pdf?url=...&format=A4&landscape=false
```

**수요처:** OG 이미지 자동 생성, 웹→PDF 변환(송장·리포트·견적서), 링크 미리보기 썸네일, 웹사이트 아카이빙(법적 증빙)

| 플랜 | 월 요금 | 캡처 수 |
|------|--------|--------|
| Free | 0원 | 100 |
| Basic | 12,000원 | 5,000 |
| Pro | 45,000원 | 40,000 |

**원가:** 캡처는 렌더링까지 필요해 텍스트 추출보다 3~5배 무겁지만, 4만 캡처 기준 약 1만원 **(마진 약 75%)**

```javascript
app.get('/shot', async (req, res) => {
  const { url, width = 1280, height = 720 } = req.query;
  const out = `/tmp/${crypto.randomUUID()}.png`;

  await run('obscura', [
    'fetch', url, '--screenshot', out,
    '--wait-until', 'networkidle0', '--quiet',
  ], { env: { ...process.env, OBSCURA_SHOT_W: width, OBSCURA_SHOT_H: height } });

  res.type('png').sendFile(out, () => fs.unlink(out, () => {}));
});
```

**리스크 (가장 중요):** Obscura는 자체 렌더링 엔진이라 복잡한 사이트에서 크롬과 픽셀이 다를 수 있습니다.
**"픽셀 퍼펙트 보장"을 팔면 안 됩니다.** 대신 "빠르고 저렴하며 대부분의 사이트에서 충분히 정확한 캡처"로 포지셔닝하고,
판매 전에 `render-repros/`의 비교 도구로 타깃 사이트 100개를 크롬과 대조 검증하세요.

---

### ④ 웹 변경 감지 알림 봇 — 난이도 ⭐⭐ / 첫 매출 4~6주

| 용도 | 타겟 |
|------|------|
| 채용공고 신규 등록 | 취준생 |
| 수강신청 자리 알림 | 대학생 |
| 한정판 드랍 / 재입고 | 리셀러 |
| 공공기관 공고 (조달청·지자체) | 프리랜서·중소기업 |
| 아파트 청약 / 부동산 매물 | 일반인 |
| 논문·특허 신규 등록 | 연구자 |

| 플랜 | 월 요금 | 감시 URL | 주기 |
|------|--------|---------|------|
| Free | 0원 | 3개 | 6시간 |
| Plus | 4,900원 | 30개 | 10분 |
| Pro | 14,900원 | 200개 | 3분 |

**원가:** 변경 감지는 해시 비교라 텍스트만 뽑으면 초경량. 유저당 약 300원 **(마진 약 94%)**

```javascript
import crypto from 'node:crypto';

async function checkChange(watch) {
  const args = ['fetch', watch.url, '--dump', 'text', '--quiet'];
  if (watch.selector) args.push('--selector', watch.selector);

  const { stdout } = await run('obscura', args);
  const hash = crypto.createHash('sha256').update(stdout).digest('hex');

  if (watch.lastHash && watch.lastHash !== hash) {
    await notify(watch, diffSummary(watch.lastText, stdout));
  }
  return { hash, text: stdout };
}
```

**차별점:** Visualping·Distill 같은 해외 서비스는 한국 사이트에서 자주 깨집니다. 카카오톡 알림 지원이 큰 무기입니다.

**리스크:** B2C는 이탈률이 높아 알림 품질(오탐 최소화)이 생명. **매크로/자동구매로 확장하면 안 됩니다 — 알림까지만.**

---

### ⑤ 노코드 커넥터 (n8n / Make / Zapier) — 난이도 ⭐⭐

아직 Obscura 노드가 없어 **선점 효과**가 있습니다. 직접 수익보다 **①번으로 유입시키는 마케팅 채널**로 보는 것이 맞습니다.

```typescript
export class Obscura implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Obscura',
    name: 'obscura',
    group: ['transform'],
    properties: [
      { displayName: 'URL', name: 'url', type: 'string', default: '' },
      { displayName: 'Operation', name: 'op', type: 'options',
        options: [
          { name: 'Get Markdown', value: 'markdown' },
          { name: 'Get Text',     value: 'text' },
          { name: 'Screenshot',   value: 'screenshot' },
          { name: 'Evaluate JS',  value: 'eval' },
        ], default: 'markdown' },
      { displayName: 'Stealth', name: 'stealth', type: 'boolean', default: true },
    ],
  };
}
```

전략: 노드는 무료 오픈소스로 배포 → 인지도 확보 → 노드 안에서 호스팅 API 안내 → 유료 전환.

---

### ⑥ 니치 데이터 판매 — 난이도 ⭐⭐⭐⭐ / 첫 매출 8~16주

툴이 아니라 **툴로 만든 데이터**를 파는 모델. B2B라 객단가가 10배 이상 높습니다.

| 데이터 상품 | 구매자 | 가격대 |
|------------|--------|-------|
| 업종별 채용공고 통합 DB (주간 갱신) | HR테크, 리서치사 | 월 30~100만원 |
| 공공입찰 공고 통합 알림 | 중소기업, 프리랜서 | 월 3~10만원 |
| 카테고리별 가격 지수 | 유통사, 애널리스트 | 월 50~200만원 |
| 스타트업 채용 동향 리포트 | VC, 언론 | 건당 50~300만원 |

**리스크 (가장 큼):** 데이터 재판매는 원본 사이트 약관 위반 가능성이 높습니다.
반드시 **공개 데이터 + 재가공(원본 그대로가 아닌 통계·지수 형태)** 으로 가야 하며,
**공공데이터포털(data.go.kr) 등 공식 개방 데이터** 중심으로 시작하는 것이 가장 안전합니다.

---

### 8-1. 한눈에 비교

| # | 아이디어 | 난이도 | 예상 마진 | 첫 매출 | 경쟁 | 법적 리스크 |
|---|---------|-------|----------|--------|------|-----------|
| ① | 웹 리더 API | ⭐ | 55~90% | 2~4주 | 높음 | 🟢 낮음 |
| ② | 가격 모니터링 | ⭐⭐⭐ | 68% | 6~10주 | 중간 | 🟡 중간 |
| ③ | 스크린샷 API | ⭐⭐ | 75% | 3~5주 | 중간 | 🟢 낮음 |
| ④ | 변경 감지 봇 | ⭐⭐ | 94% | 4~6주 | 낮음 | 🟢 낮음 |
| ⑤ | 노코드 커넥터 | ⭐⭐ | (깔때기) | — | 없음 | 🟢 낮음 |
| ⑥ | 데이터 판매 | ⭐⭐⭐⭐ | 매우 높음 | 8~16주 | 낮음 | 🔴 높음 |

### 8-2. 추천 전략: ①+③ 묶어 시작 → ④로 확장

**1~4주차 (MVP)**
- ① 리더 API + ③ 스크린샷 API를 하나의 서비스로 출시 (둘 다 "URL → 결과" 구조라 코드 90% 공유)
- 무료 티어를 넉넉하게 (원가가 낮아 가능한 전략)
- 랜딩페이지 1장 + 문서 + 결제 연동

**5~8주차 (유입)**
- ⑤ n8n 커넥터 배포 (npm, n8n 커뮤니티)
- 벨로그 / GeekNews / Reddit r/webscraping에 기술 글 작성
- "한국 사이트 잘 긁힘"을 전면에 배치

**9~16주차 (확장)**
- ④ 변경 감지 봇을 같은 인프라 위에 추가
- 반응에 따라 ② 또는 ⑥으로 심화

**초기 투자**

| 항목 | 비용 |
|------|------|
| 서버 | 월 2~5만원 |
| 도메인 | 연 2만원 |
| 프록시 (초기 생략 가능) | 월 0~5만원 |
| **합계** | **월 3~10만원** |

→ 유료 고객 10명이면 손익분기.

### 8-3. 솔직하게: 실패할 수 있는 이유

1. **"원가가 싸다"만으로는 안 팔립니다.** 고객은 원가를 모르고 문제 해결을 삽니다. 명확한 차별점(예: 한국 사이트 특화)이 필요합니다.
2. **유지보수가 진짜 일입니다.** 사이트가 개편되면 파서가 깨집니다. 셀렉터를 고객이 직접 고칠 수 있는 UI를 제공하는 것이 답입니다.
3. **v0.1.0 의존성 리스크.** 업스트림이 방향을 틀거나 멈출 수 있습니다. 추상화 레이어를 두어 최악의 경우 Playwright로 전환 가능하게 설계하세요.
4. **결국 마케팅 싸움입니다.** 코드는 4주면 되지만 고객 확보는 6개월 걸립니다. ⑤번 커넥터, 오픈소스 기여, 기술 블로그로 미리 인지도를 쌓아두세요.

---

## 9. React / PHP에서 쓸 수 있을까?

### 9-1. "Obscura 같은 걸 React/PHP로 만들 수 있나?" → 사실상 불가능

| 필요 요소 | React/PHP로 가능? |
|-----------|------------------|
| V8 엔진 직접 임베딩 | ❌ (C++ 레벨 작업) |
| CSS 레이아웃 엔진 (7만 줄) | 이론상 가능하나 수년 소요 |
| 메모리 30MB | ❌ 런타임 자체가 그 이상 |
| 픽셀 단위 래스터라이징 | ❌ 성능 부족 |

Rust를 쓴 이유가 **GC 없이 30MB**를 달성하기 위해서입니다. React는 UI 라이브러리, PHP는 서버 스크립트로 목적 자체가 다릅니다.

### 9-2. "React/PHP 프로젝트에서 Obscura를 쓸 수 있나?" → 100% 가능

Obscura는 서버로 띄워놓고 어떤 언어에서든 호출하는 구조입니다.

```
[ React ]  [ PHP ]  [ Python ]  [ 무엇이든 ]
    └────────┴─────────┴───────────┘
                   │
                   ▼
           [ Obscura 서버 ]
```

#### React (Node 백엔드 경유)

```javascript
// server.js
import express from 'express';
import puppeteer from 'puppeteer-core';

const app = express();

app.get('/api/scrape', async (req, res) => {
  const browser = await puppeteer.connect({
    browserWSEndpoint: 'ws://127.0.0.1:9222/devtools/browser',
  });
  const page = await browser.newPage();
  await page.goto(req.query.url, { waitUntil: 'networkidle0' });

  const data = await page.evaluate(() => ({
    title: document.title,
    text: document.body.innerText.slice(0, 2000),
  }));

  await browser.disconnect();
  res.json(data);
});

app.listen(3001);
```

```jsx
// App.jsx
function Scraper() {
  const [data, setData] = useState(null);

  const scrape = async (url) => {
    const r = await fetch(`/api/scrape?url=${encodeURIComponent(url)}`);
    setData(await r.json());
  };

  return (
    <div>
      <button onClick={() => scrape('https://example.com')}>긁어오기</button>
      {data && <pre>{data.title}</pre>}
    </div>
  );
}
```

#### PHP — 방법 1: CLI 직접 호출 (가장 간단)

```php
<?php
function scrape(string $url): array {
    $cmd = sprintf(
        'obscura fetch %s --dump markdown --stealth --quiet',
        escapeshellarg($url)
    );
    return ['url' => $url, 'content' => shell_exec($cmd)];
}

function evaluate(string $url, string $js): string {
    $cmd = sprintf(
        'obscura fetch %s --eval %s --quiet',
        escapeshellarg($url),
        escapeshellarg($js)
    );
    return trim(shell_exec($cmd));
}

echo evaluate('https://example.com', 'document.title');
```

> ⚠️ `escapeshellarg()`를 반드시 사용하세요. 없으면 명령어 주입(command injection) 취약점이 생깁니다.

#### PHP — 방법 2: MCP HTTP 엔드포인트 호출

```bash
obscura mcp --http --port 3000
```

```php
<?php
function mcpCall(string $tool, array $args): array {
    $payload = json_encode([
        'jsonrpc' => '2.0',
        'id'      => 1,
        'method'  => 'tools/call',
        'params'  => ['name' => $tool, 'arguments' => $args],
    ]);

    $ch = curl_init('http://127.0.0.1:3000/mcp');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        CURLOPT_POSTFIELDS     => $payload,
    ]);
    $res = curl_exec($ch);
    curl_close($ch);

    return json_decode($res, true);
}

mcpCall('browser_navigate', ['url' => 'https://example.com']);
print_r(mcpCall('browser_snapshot', ['max_chars' => 3000]));
```

#### PHP — 방법 3: CDP WebSocket 직접 연결

`textalk/websocket` 같은 라이브러리로 `ws://127.0.0.1:9222`에 직접 붙는 방식. 가장 강력하지만 구현 난도가 높습니다.

#### Rust — 라이브러리로 임베딩

```rust
use obscura::Browser;
use std::time::Duration;

let browser = Browser::builder()
    .stealth(true)
    .storage_dir("/tmp/obscura-data")
    .build()?;

let mut page = browser.new_page().await?;
page.goto("https://example.com").await?;
let el = page.wait_for_selector("a", Duration::from_secs(5)).await?;
println!("{:?}", el.attribute("href"));
```

---

## 10. 시작 전 체크리스트

| 항목 | 설명 |
|------|------|
| **Apache-2.0 준수** | 상업적 사용 가능. 단 ⓐ라이선스 사본 포함 ⓑ변경사항 명시 ⓒ저작권 고지 유지 |
| **robots.txt 존중** | `--obey-robots` 켜기 (기본값 off) |
| **대상 사이트 약관 확인** | 특히 가격 모니터링·데이터 판매 |
| **개인정보 수집 금지** | 국내 개인정보보호법 위반은 형사처벌 대상. 공개 상품정보·공고문만 |
| **요청 속도 조절** | 상대 서버 부담은 업무방해 소지. 딜레이 필수 |
| **MCP HTTP 노출 주의** | 인증 없음. `OBSCURA_MCP_ALLOWED_ORIGINS` + 리버스 프록시 필수 |
| **렌더링 검증** | 스크린샷 상품은 `render-repros/` 도구로 크롬과 대조 검증 |
| **버전 고정** | 아직 v0.1.0. 프로덕션 전 충분히 테스트하고 버전 핀 고정 |

---

## 요약

| 질문 | 답 |
|------|-----|
| 설치 | 도커 1줄, 또는 `-stealth` 바이너리 다운로드 |
| 정체 | 독립 실행 프로그램 + MCP 서버 내장 + 스킬 문서 동봉 |
| API 토큰 | 불필요. 완전 무료, 로컬 실행 |
| 인기 이유 | AI 붐 타이밍 + 크로미움 탈출 + Cloudflare 레퍼런스 + 낮은 진입장벽 |
| 에이전트 활용 | 최적. 킬러 유즈케이스 |
| 수익화 | 마크다운 리더 API가 가장 쉽고 빠름 (원가 경쟁력 압도적) |
| React/PHP | 엔진을 만들 순 없지만, 서버로 띄우고 HTTP/CLI로 호출 가능 |
