# 테스트 전략

## Vitest + Playwright 표준 조합

프론트엔드 테스트는 Vitest(단위/통합)와 Playwright(E2E)로 역할을 나눈다.

| 계층 | 도구 | 실행 환경 | 시나리오 정의 방식 |
|------|------|----------|------------------|
| 단위 | Vitest (node) | Node.js | 코드 API 호출 |
| 통합 | Vitest (jsdom) | Node.js + jsdom | 코드 API 호출 (TestEditor 등) |
| E2E | Playwright | 실제 브라우저 | 유저 행동 (클릭, 드래그, 타이핑) |

단위/통합은 **코드 API로** 시나리오를 기술하고, E2E부터는 **유저 인터랙션으로** 시나리오를 기술한다.

```ts
// 통합 — 내부 API로 시뮬레이션
editor.createShapes([{ type: 'geo', ... }])
editor.select(ids.box1)
editor.pointerDown(60, 10, { handle: 'top_left_rotate' })

// E2E — 유저 행동 그대로
await page.mouse.click(100, 100)
await page.mouse.move(200, 200)
```

이전에는 Jest + Cypress 조합이 주류였으나, Vitest가 빠르고(Vite 기반 ESM 네이티브) Playwright가 안정적이어서(멀티브라우저) 현재 사실상 표준.

## Vitest 멀티프로젝트 구조

루트 `vitest.config.ts`가 모든 워크스페이스의 설정을 glob으로 수집하여 멀티프로젝트로 실행한다.

> 📍 vitest.config.ts

```ts
const vitestPackages = glob.sync('./{apps,packages}/**/vitest.config.ts')
export default defineConfig({
  test: { projects: vitestPackages },
})
```

총 24개 워크스페이스가 각자의 `vitest.config.ts`를 가지며, 루트에서 `yarn vitest`하면 전체 실행(느림), 보통 개별 워크스페이스에서 `yarn test` 실행.

## Vitest 실행 원리

Vitest는 테스트 러너이자 Vite의 변환 파이프라인을 재사용하는 프레임워크다.

```
vitest CLI (메인 Node 프로세스)
  ├─ Vite Dev Server — 변환 파이프라인 (TS, JSX, SVG 등)
  └─ Worker Thread(s) — 테스트 실행
       ├─ environment 설정에 따라 jsdom 등을 로드
       └─ 테스트 코드 import → 실행 → 결과 보고
```

- 기본 `pool: 'threads'` — Node.js `worker_threads` (경량 병렬화)
- 옵션 `pool: 'forks'` — `child_process.fork()` (별도 프로세스, Node 플래그 전달 필요 시)
- Vite 변환 결과를 캐싱하고, watch 모드에서는 변경된 파일만 재실행 → Jest보다 빠른 이유

## Vitest와 jsdom의 관계

Vitest와 jsdom은 완전 별도 프로젝트. Vitest는 테스트 러너, jsdom은 브라우저 환경 시뮬레이터.

```ts
test: {
  environment: 'jsdom'    // Node global에 window, document 등을 심어놓음
  // environment: 'node'  // 순수 Node 환경
}
```

`environment: 'jsdom'`이면 Vitest가 테스트 실행 전에 jsdom 인스턴스를 만들어 Node의 global을 브라우저처럼 꾸며놓는 것. 테스트 코드는 마치 브라우저 안에서 돌아가는 것처럼 `document.createElement()` 등을 쓸 수 있게 된다.

단, jsdom은 순수 JS로 DOM만 흉내내는 프로젝트라 **네이티브 엔진이 필요한 API**(Canvas, requestAnimationFrame, Image.decode, CSS.supports, 레이아웃 등)는 구현되어 있지 않다. 이 빈 구멍을 프로젝트가 직접 폴리필로 메꿔야 한다.

## 베이스 프리셋 (`node-preset.ts`)

대부분의 워크스페이스가 상속하는 공통 Vitest 설정.

> 📍 internal/config/vitest/node-preset.ts

```ts
export default defineConfig({
  plugins: [svgTransform],         // .svg → export default {} (빈 객체로 치환)
  test: {
    globals: true,                 // describe, it, expect import 없이 전역 사용
    environment: 'jsdom',
    setupFiles: ['./setup.ts'],    // 공통 폴리필/모킹
    include: ['src/**/*.{test,spec}.{js,ts,jsx,tsx}'],
    exclude: ['__fixtures__', 'node_modules', 'dist', ...],
    coverage: { ... },
  },
  resolve: {
    alias: { '~': 'src/' },       // ~/utils → src/utils
  },
  ssr: {
    noExternal: ['@tiptap/react'], // ESM/CJS 호환 문제로 번들링에 포함
  },
})
```

**svgTransform 플러그인**: 테스트에서 `.svg` import 시 Node가 SVG를 JS 모듈로 해석할 수 없으므로, 빈 객체로 치환하여 import 에러를 방지. 테스트에서 아이콘의 실제 내용은 불필요하기 때문.

## 글로벌 셋업 (`setup.ts`)

Node에서 브라우저인 척 할 수 있게 가짜 API를 채우고, tldraw 특유의 부동소수점 비교 매처를 추가하는 파일.

> 📍 internal/config/vitest/setup.ts

### 브라우저 API 폴리필

jsdom이 제공하지 않는 API들을 가짜로 채워넣는다. 일반 웹앱은 이런 API를 잘 안 쓰지만, tldraw는 캔버스 기반 그래픽 앱이라 저수준 렌더링 API에 의존하기 때문.

| 폴리필 | 이유 |
|--------|------|
| `vitest-canvas-mock` | Canvas 2D API (jsdom에 렌더링 엔진 없음) |
| `requestAnimationFrame` → `setTimeout(cb, 16)` | 화면 갱신 루프 (jsdom에 디스플레이 없음) |
| `TextEncoder/TextDecoder` | Node util에서 가져와 전역에 심음 |
| `crypto` → Node crypto / `@peculiar/webcrypto` | ID 생성 등에 사용 |
| `Image.decode()` → `Promise.resolve()` | 이미지 디코더 없으므로 즉시 resolve |
| `Path2D.roundRect` → `this.rect()` | 둥근 모서리 무시, 일반 rect로 대체 |
| `CSS.supports` → `() => false` | CSS 엔진 없으므로 항상 미지원 반환 |

### 커스텀 매처: `toCloselyMatchObject`

캔버스 좌표/크기 계산에서 부동소수점 오차를 허용하는 비교 매처.

```ts
// 회전 후 좌표가 99.99999999 같은 값이 될 수 있음
expect({ x: 99.99999999, y: 200.0000001 })
  .toCloselyMatchObject({ x: 100, y: 200 })  // ✅ 통과
```

actual과 expected의 모든 숫자를 `roundToNearest`(기본 0.0001) 단위로 반올림한 뒤 비교.

## 워크스페이스별 설정 차이

각 워크스페이스는 베이스 프리셋을 상속하고 필요에 따라 확장한다.

| 패키지 | 환경 | 추가 설정 |
|--------|------|----------|
| editor | jsdom | PointerEvent/DragEvent/TouchEvent 모킹, fake-indexeddb, ResizeObserver, fake timers |
| tldraw | jsdom | 번역/아이콘 CDN fetch 모킹, breakpoints 컨텍스트 모킹, DOMRect 폴리필, fake timers |
| store | node | crypto 폴리필만 |
| state, validate, tlschema | node | 커스텀 설정 없음 (순수 JS) |
| sync-worker | node | `pool: 'forks'`, `--experimental-sqlite` 플래그 |
| dotcom/client | jsdom | 독립 설정(프리셋 미사용), timeout 30초, 환경변수 |

DOM 의존도에 따라 분리: 순수 로직 패키지는 node 환경, UI 관련은 jsdom + 브라우저 API 모킹.

## Playwright E2E 설정 상세

### examples Playwright 설정

> 📍 apps/examples/e2e/playwright.config.ts

```ts
const config: PlaywrightTestConfig = {
  globalSetup: './global-setup.ts',       // no-op (빈 함수)
  globalTeardown: './global-teardown.ts',
  timeout: 30 * 1000,                     // 테스트당 30초
  expect: {
    timeout: 2000,                         // expect 대기 2초
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.0001,           // 매우 엄격한 스크린샷 비교
      threshold: 0.01,
    },
  },
  fullyParallel: true,                     // 최대 병렬화
  retries: process.env.CI ? 1 : 0,         // CI에서만 1회 재시도
  reporter: process.env.CI
    ? [['list'], ['github'], ['html', { open: 'never' }]]
    : 'list',
  use: {
    trace: 'on-first-retry',               // 재시도 시에만 trace 수집
    headless: true,
    video: 'retain-on-failure',            // 실패 시에만 비디오 유지
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Chrome', use: { ...devices['Pixel 5'] } },
  ],
  webServer: [
    { command: 'yarn preview', port: 5420 },  // 예제 앱
    { command: 'yarn dev', port: 8989 },      // bemo-worker (멀티플레이어)
  ],
}
```

핵심 설계:
- 데스크톱 + 모바일(Pixel 5) 두 프로젝트로 반응형 테스트
- 스크린샷 비교가 매우 엄격 (`maxDiffPixelRatio: 0.0001`) — 픽셀 단위 시각적 회귀 감지
- CI에서 3중 리포터: `list`(콘솔), `github`(PR 어노테이션), `html`(상세 리포트)

### dotcom Playwright 설정

> 📍 apps/dotcom/client/playwright.config.ts

```ts
export default defineConfig({
  testDir: './e2e',
  fullyParallel: process.env.STAGING_TESTS ? true : false,  // 기본 순차
  retries: process.env.CI ? 2 : 0,                          // CI에서 2회 재시도
  workers: process.env.STAGING_TESTS ? 6 : process.env.CI ? 2 : 3,
  projects: [
    { name: 'global-setup', testMatch: /global\.setup\.ts/ },
    { name: 'chromium', use: { ...devices['Desktop Chrome'] }, dependencies: ['global-setup'] },
    { name: 'staging', use: { storageState: 'e2e/.auth/staging.json' }, dependencies: ['global-staging-setup'] },
  ],
  webServer: {
    command: process.env.CI ? 'VITE_PREVIEW=1 yarn dev-app' : 'yarn preview-app',
    url: 'http://localhost:3000',
  },
})
```

examples와의 주요 차이:
- **Clerk 인증 필요**: `global-setup`에서 `@clerk/testing/playwright`의 `clerkSetup()` 호출
- **프로젝트 의존성 체인**: `global-setup` → `chromium` (셋업 완료 후 테스트)
- **`fullyParallel: false`**: 클립보드 같은 공유 시스템 리소스를 사용하는 테스트가 있어 파일 내 순차 실행
- **workers 2개 (CI)**: DB 커넥션 충돌 방지
- **staging 프로젝트**: 별도 인증 상태(`staging.json`)로 스테이징 환경 테스트

### E2E 스크립트 매핑

```
yarn e2e           → playwright test ./e2e/tests/        (examples 기능 테스트)
yarn e2e-perf      → playwright test ./e2e/perf/         (examples 성능 테스트)
yarn e2e-dotcom    → playwright test --project=chromium   (dotcom 테스트)
yarn e2e-dotcom-x10 → playwright test --repeat-each=10   (dotcom 10회 반복, flaky 탐지)
```

> 📍 관련 인프라: `.study/01-dev-infra.md` — Playwright E2E 워크플로우 (4개)
