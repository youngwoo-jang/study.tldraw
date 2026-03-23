# 개발 인프라 & 설정

## 오픈소스 루트 마크다운 파일 컨벤션

GitHub이 자동 인식하는 표준 파일들을 프로젝트 규모에 맞게 배치한다.

> 📍 / (리포지토리 루트)

**필수 (모든 프로젝트)**
- `README.md` — GitHub이 자동 렌더링. 없으면 미완성 프로젝트로 인식
- `LICENSE.md` — 없으면 법적으로 "모든 권리 보유" 상태. 리포 상단에 배지 자동 표시

**준필수 (기여자를 받는 프로젝트)**
- `CONTRIBUTING.md` — GitHub이 PR/이슈 작성 시 링크 자동 노출
- `CODE_OF_CONDUCT.md` — Community Profile 체크리스트 항목
- `SECURITY.md` — Security 탭에서 자동 인식. 취약점 신고 채널 안내

**기업 오픈소스 추가**
- `CLA.md` — 기여자 라이선스 동의서 (법적 보호)
- `TRADEMARKS.md` — 브랜드/상표 보호 (tldraw처럼 상용 제품이 있을 때)

**선택적**
- `RELEASES.md`, `FAQ.md`, `COMPANY.md` — 프로젝트 상황에 따라

tldraw는 기업 운영 상용 오픈소스라 풀세트(`README`, `LICENSE`, `CONTRIBUTING`, `CODE_OF_CONDUCT`, `SECURITY`, `CLA`, `TRADEMARKS`, `COMPANY`, `FAQ`, `RELEASES`)를 갖추고 있다.

## Yarn Berry 설정 (`.yarnrc.yml`)

PnP 대신 `node_modules` 방식을 채택하고, 프로젝트별 캐시를 사용한다.

> 📍 .yarnrc.yml

```yaml
compressionLevel: mixed        # 텍스트는 압축, 바이너리는 무압축 (속도/용량 균형)
enableGlobalCache: false        # 프로젝트별 .yarn/cache/에 독립 저장 (CI 편의성)
enableInlineBuilds: true        # 네이티브 빌드 로그를 터미널에 실시간 출력
nodeLinker: node-modules         # PnP 대신 전통적 node_modules 방식
```

**PnP를 안 쓰는 이유**: Vite, Next.js, Wrangler, esbuild, API Extractor 등 도구 호환성 문제가 많아서. 대규모 모노레포에서 흔한 실용적 선택.

**`enableGlobalCache: false`인 이유**: 글로벌 캐시(`~/.yarn/berry/cache/`)와 로컬 캐시(`.yarn/cache/`) 중 로컬 선택. 캐시는 다운로드 원본 zip 보관소일 뿐이라 버전 충돌과는 무관. CI에서 프로젝트 디렉토리만 캐싱하면 되는 편의성 정도의 차이.

## Yarn Constraints (`yarn.config.cjs`)

모노레포 전체 `package.json`들의 의존성 버전을 자동으로 통일시키는 룰 엔진.

> 📍 yarn.config.cjs

```js
function enforceConsistentDependenciesAcrossTheProject({ Yarn }) {
  // 같은 패키지를 쓰는 모든 워크스페이스의 버전 범위를 동기화
  for (const dependency of Yarn.dependencies()) {
    if (dependency.type === 'peerDependencies') continue
    for (const otherDependency of Yarn.dependencies({ ident: dependency.ident })) {
      if (otherDependency.type === 'peerDependencies') continue
      dependency.update(otherDependency.range)
    }
  }
  // peer deps도 별도 루프로 동일하게 처리
  // ...
  // 하위 워크스페이스에서 packageManager 필드 제거 (루트에만 존재해야 함)
  for (const workspace of Yarn.workspaces()) {
    if (workspace.cwd === '.') continue
    workspace.set('packageManager', undefined)
  }
}
```

- `yarn constraints` — 위반 사항 체크, `yarn constraints --fix` — 자동 수정
- CI(`.github/workflows/checks.yml`)에서 실행. husky pre-commit에는 없음 (무거우니까 CI에서 한번 잡는 전략)
- **Yarn Berry 전용 기능**. npm/pnpm에는 없고, 비슷한 걸 하려면 syncpack이나 manypkg 같은 외부 도구 필요

## `package.json` 핵심 설정

모노레포의 중앙 설정. 워크스페이스, 버전 고정, 의존성 오버라이드를 관리한다.

> 📍 package.json

**`workspaces`** — 모노레포 워크스페이스 경로. `apps/dotcom/*`처럼 2뎁스인 건 하위에 client, sync-worker 등 여러 패키지가 있기 때문.

**`packageManager: "yarn@4.12.0"`** — Corepack과 연동. `corepack enable` 후 이 프로젝트에서는 정확히 이 버전의 yarn만 사용 가능. npm이나 다른 버전 yarn 실행 시 에러. 모노레포에서는 거의 필수.

**`engines: { node: "^20.0.0" }`** — Node 20.x만 허용. 다른 버전이면 경고/에러.

**`lint-staged`** — staged된 파일만 oxfmt로 자동 포매팅. husky pre-commit → lint-staged → oxfmt 체인. husky(git hook 등록), lint-staged(staged 파일 필터), oxfmt(포매팅)는 각각 독립 패키지.

**`resolutions`** — 의존성 트리의 패키지를 강제 대체.
- `canvas → empty-npm-package`: pdf.js가 끌고 오는 네이티브 C++ 패키지를 빈 패키지로 교체 (브라우저에서 불필요, 설치 시간 단축)
- `@microsoft/tsdoc → patch:`: `.yarn/patches/`의 diff를 적용하여 원본 패키지 수정
- `@types/node → ^22.15.31`: 의존성 트리 어디서든 동일 버전으로 강제 통일

**`postinstall: "husky install && yarn refresh-assets"`** — `yarn install` 완료 후 자동 실행되는 예약 스크립트. git hooks 등록 + 에셋 파일 생성을 자동화하여 별도 셋업 명령 불필요.

## LazyRepo 빌드 시스템

tldraw 공동 창업자 David Sheldrick(ds300)이 만든 모노레포 빌드 오케스트레이터. 증분 빌드 + 캐싱으로 변경된 패키지만 빌드한다.

> 📍 lazy.config.ts
> 📍 internal/scripts/build-package.ts

### 빌드 실행 체인

yarn은 스크립트 실행기일 뿐, 실제 빌드는 esbuild가 수행한다.

```
yarn build
  → lazy build (lazyrepo: 순서/캐싱/병렬화 관리)
    → 각 패키지의 package.json scripts.build
      → build-package.ts → esbuild (TS→JS 변환)
```

> 📍 관련 개념: `.study/00-dev-concepts.md` — 모노레포 오케스트레이터

### 태스크 구성

| 태스크 | 실행 방식 | 설명 |
|--------|----------|------|
| `build` | 워크스페이스별 | prebuild, refresh-assets, build-i18n 후 실행 |
| `dev` | independent, 캐시 없음 | 개발 서버 (매번 실행) |
| `refresh-assets` | top-level (1회) | assets/ 변경 감지하여 아이콘/폰트/번역 재생성 |
| `build-types` | top-level (1회) | typecheck.ts로 전체 타입 빌드 |
| `build-api` | independent | .tsbuild/*.d.ts → api/* 생성 |
| `api-check` | top-level (1회) | public.d.ts 기반 API 일관성 검증 |

### 의존성 그래프

```
refresh-assets ──┐
                 ├──→ build-types ──→ build-api ──→ api-check
maybe-clean-     │
tsbuildinfo ─────┘
prebuild ────────────────────────→ build (packages/*)
refresh-assets ──────────────────→ build (apps/*)
build-i18n ──────────────────────→ build
```

### build-api 파이프라인 상세

`build-types` 이후 각 패키지에서 독립 실행되는 API 추출 파이프라인.

```
.tsbuild/          ← tsc가 생성한 원본 .d.ts + .js (캐시 대상)
    ↓ cp -R
.tsbuild-api/      ← 사본 (원본 캐시 보호용)
    ↓ sortUnions (recast AST로 union 멤버 알파벳순 정렬)
.tsbuild-api/      ← 정렬된 .d.ts
    ↓ api-extractor run
    ├── api/public.d.ts        ← @public만 (api-check 검증용)
    ├── api/internal.d.ts      ← 전체 (참조용)
    ├── api/api.json           ← docs 사이트 API 레퍼런스 생성용
    └── api-report.api.md      ← git 추적, PR에서 API 변경 감지
```

- `.tsbuild-api`는 `.tsbuild`의 복사본. sortUnions로 수정해야 하는데 원본을 건드리면 lazyrepo 캐시가 깨지므로 사본에서 작업
- sortUnions: tsc가 union 멤버 순서를 비결정적으로 출력하기 때문에 알파벳순 정렬로 항상 동일한 출력 보장
- CI에서는 `api-extractor run` (엄격 — api-report 변경 시 에러), 로컬에서는 `--local` (자동 업데이트)

> 📍 internal/scripts/build-api.ts
> 📍 internal/scripts/lib/sort-unions.ts
> 📍 internal/config/api-extractor.json
> 📍 상세 분석: `.study/05-api-design.md` — TSDoc + API Extractor 파이프라인

### `usesOutput: false` 최적화

`build-types`는 top-level이라 출력이 모든 워크스페이스의 `.tsbuild/`. `build-api`가 `usesOutput: true`(기본값)면 모든 패키지의 `.tsbuild`에 의존하게 된다. 실제로는 자기 패키지의 `.tsbuild/*.d.ts`만 필요하므로 `usesOutput: false`로 끊고, `cache.inputs`에서 로컬 `.tsbuild/**/*.d.ts`만 참조하게 설정. 이렇게 하지 않으면 다른 패키지의 `.d.ts`가 바뀔 때마다 무관한 패키지의 `build-api`도 재실행된다.

```ts
'build-api': {
  execution: 'independent',
  cache: {
    inputs: ['.tsbuild/**/*.d.ts', 'tsconfig.json'],  // 자기 것만
  },
  runsAfter: {
    'build-types': { usesOutput: false },  // 전체 출력 의존성 차단
  },
}
```

### esbuild 설정

별도 설정 파일 없이 `build-package.ts` 스크립트가 설정과 빌드 로직을 겸한다.

```ts
await build({
  entryPoints: sourceFiles,
  outdir,
  format: info.moduleSystem,   // 'cjs' 또는 'esm'
  bundle: false,               // 번들링 안 함 (파일별 변환만)
  platform: 'neutral',
  sourcemap: true,
  target: 'es2022',
})
```

- `bundle: false`: SDK이므로 번들링하지 않음. 트리 쉐이킹을 사용자에 맡김
- CJS(`dist-cjs/`) + ESM(`dist-esm/`) 두 벌 생성
- `api/public.d.ts`를 각 dist에 복사하여 타입 정의 배포

## Lerna + 커스텀 퍼블리싱 인프라

Lerna는 "현재 버전 기록"용으로만 사용하고, 실제 버전 범프와 퍼블리싱은 커스텀 TypeScript 스크립트 + GitHub Actions가 담당한다.

> 📍 lerna.json
> 📍 internal/scripts/lib/publishing.ts
> 📍 .github/workflows/publish-*.yml

### Lerna 설정

```json
{ "packages": ["packages/*"], "version": "4.5.3" }
```

- Fixed 모드: 17개 SDK 패키지 전부 동일 버전 (4.5.3)
- `apps/`는 제외 — npm 배포 대상이 아님
- Lerna CLI(`lerna publish` 등)는 직접 사용하지 않음

### 퍼블리싱 워크플로우

| 워크플로우 | 트리거 | npm dist tag |
|---|---|---|
| `publish-new.yml` | 수동, `production` 브랜치 | `latest` |
| `publish-patch.yml` | `v*.*.x` 브랜치 push 시 자동 | `latest` / `revision` |
| `publish-canary.yml` | `main` push 시 자동 | `canary` |
| `publish-canary.yml` | `production` push 시 자동 | `next` |
| `publish-manual.yml` | 수동 | 실패 복구용 재퍼블리시 |
| `bump-versions.yml` | 릴리스 후 | main 브랜치 버전 동기화 |

### 핵심 유틸리티 (`lib/publishing.ts`)

- `setAllVersions(version)` — 모든 package.json + lerna.json 버전 일괄 변경
- `publish(distTag)` — 의존성 토폴로지컬 정렬 순서로 npm 퍼블리시 + 재시도 로직
- `getLatestTldrawVersionFromNpm()` — npm 레지스트리에서 최신 버전 조회

### 릴리스 브랜치 전략

```
main ──push──→ canary (4.5.3-canary.{sha})
production ──수동──→ minor/major 릴리스 (4.6.0)
                      ├→ 릴리스 브랜치 v4.6.x 생성
                      └→ bump-versions로 main 동기화
v*.*.x ──push──→ 자동 패치 릴리스 (4.6.1)
```

> 📍 관련 개념: `.study/00-dev-concepts.md` — npm 배포, Lerna, 모듈 분리 전략

## Vitest 테스트 설정

단위/통합 테스트에 Vitest를 사용한다. 루트 설정이 모든 워크스페이스의 설정을 수집하여 멀티프로젝트로 실행.

> 📍 vitest.config.ts
> 📍 internal/config/vitest/node-preset.ts
> 📍 internal/config/vitest/setup.ts

```ts
// 루트 vitest.config.ts — 워크스페이스 자동 수집
const vitestPackages = glob.sync('./{apps,packages}/**/vitest.config.ts')
export default defineConfig({
  test: { projects: vitestPackages },
})
```

- 총 24개 워크스페이스가 각자의 `vitest.config.ts`를 가짐
- 대부분 `internal/config/vitest/node-preset.ts`를 베이스 프리셋으로 상속
- 프리셋이 공통 설정 제공: jsdom 환경, globals, SVG 변환, `~` alias 등
- `setup.ts`에서 브라우저 API 폴리필(Canvas, rAF, crypto 등)과 커스텀 매처 등록
- 각 워크스페이스는 필요에 따라 환경(node/jsdom), fake timers, 추가 모킹 등을 오버라이드

> 📍 상세 분석: `.study/08-testing.md`

## Git 관련 설정 파일 — `.gitignore`, `.ignore`, `.dockerignore`

프로젝트의 세 가지 ignore 파일은 각각 다른 도구가 읽고, 다른 목적을 가진다.

> 📍 .gitignore
> 📍 .ignore
> 📍 .dockerignore

### `.gitignore` — Git 추적 제외 (132줄)

| 카테고리 | 패턴 | 설명 |
|---------|------|------|
| **빌드 산출물** | `dist`, `dist-cjs`, `dist-esm`, `.tsbuild*`, `.lazy` | 빌드된 JS/CSS 번들, TS 빌드 캐시, LazyRepo 캐시 |
| **패키지 관리** | `node_modules`, `.pnp.*`, `.yarn/*` | Yarn Berry. `patches`, `plugins`, `releases`, `sdks`, `versions`만 `!`로 허용 |
| **에디터 설정** | `.vscode/*`, `.idea`, `.zed` | VS Code는 `extensions.json`만 허용 |
| **환경변수** | `**/*.env`, `.env*`, `.dev.vars` | 모든 환경변수 파일 (보안상 필수) |
| **생성된 에셋** | `packages/assets/{embed-icons,fonts,icons,translations,watermarks}` | `yarn refresh-assets`로 자동 생성 |
| **API 문서** | `api-json`, `api-md`, `packages/*/api/temp` | API Extractor 중간 산출물 |
| **테스트 결과** | `test-results`, `playwright-report`, `coverage` | Vitest, Playwright 결과 |
| **인프라** | `.vercel`, `.wrangler` | 배포 플랫폼 설정 |
| **AI 도구** | `.mcp.json`, `opencode.json`, `.claude/skills/pr-walkthrough/tmp` | AI 코딩 도구 설정/캐시 |

`.yarn/*`에서 `patches`, `releases` 등을 `!`로 다시 포함시키는 패턴은 Yarn Berry 표준. 패치 파일과 릴리스 바이너리는 팀 전체가 동일 버전을 쓰도록 커밋에 포함.

### `.ignore` — 검색 도구용 제외 (ripgrep/ag/fd)

`.gitignore`보다 **더 공격적으로 필터링**. Git에는 커밋하지만 검색 시 노이즈인 파일들을 제외:

- `yarn.lock` — 수천 줄의 lock 파일
- `*.d.ts`, `*.d.ts.map` — 타입 선언 (원본 `.ts`를 보면 됨)
- `**/*.tldr` — tldraw 문서 바이너리
- `**/api-report.md` — 자동 생성된 API 리포트
- `apps/docs/content/releases/**/*`, `apps/docs/content/reference/**/*` — 자동 생성 문서

VS Code의 `Cmd+Shift+F`가 내부적으로 ripgrep을 사용하므로, IDE 검색에도 자동 적용.

> 📍 관련 개념: `.study/00-dev-concepts.md` — `.ignore` 파일, Docker 빌드 컨텍스트

### `.dockerignore` — Docker 빌드 컨텍스트 제외

`.gitignore`와 거의 동일하나, `.git` 디렉토리를 명시적으로 제외하는 점이 다르다. 로컬에서 `docker build .` 시 `.git`(수백 MB)이 Docker 데몬에 전송되는 것을 방지.

CI/CD(쿠버네티스 등)에서는 `git clone`/shallow clone으로 `.git`이 없거나 작아서 실질적 필요성이 낮음. 주로 **로컬에서 `docker build`하는 개발자**를 위한 설정.

## Husky Git Hooks

husky가 4개의 Git hook을 등록하여 커밋/푸시/체크아웃/머지 시 자동 검증을 수행한다.

> 📍 .husky/pre-commit
> 📍 .husky/pre-push
> 📍 .husky/post-checkout
> 📍 .husky/post-merge

### pre-commit — 커밋 시 자동 빌드/포매팅

실행 순서:
1. **package.json 변경 감지** → `yarn install --immutable` (lock 파일 정합성 보장)
2. **packages/ 변경 감지** → 변경된 패키지만 `build-api` 실행 + `api-report.api.md` 자동 스테이징
3. **dotcom client 변경 감지** → `build-i18n` 실행 + 로케일 파일 자동 스테이징
4. **`yarn lint-staged`** → oxfmt 포매팅

핵심 설계 원칙은 **"변경된 것만 처리"**. 모노레포 전체를 매번 검증하지 않고, staged 파일 기반으로 필요한 패키지만 선택적으로 처리.

### 다른 hooks

| hook | 역할 |
|------|------|
| **pre-push** | Git LFS 검증 (대용량 파일 관리) |
| **post-checkout** | `.tsbuildinfo` 파일 전체 삭제 + yarn.lock 변경 시 알림 |
| **post-merge** | yarn.lock 변경 시 `yarn` 실행 알림 |

`post-checkout`에서 `.tsbuildinfo`를 삭제하는 이유: 브랜치 전환 시 이전 브랜치의 증분 빌드 캐시가 남아있으면 잘못된 빌드 결과가 나올 수 있음.

### 전체 흐름

```
개발자가 git commit 실행
  ↓
pre-commit (husky)
  ├─ package.json 변경? → yarn install --immutable
  ├─ packages/ 변경? → build-api → api-report 자동 스테이징
  ├─ dotcom client 변경? → build-i18n → 로케일 파일 자동 스테이징
  └─ lint-staged → oxfmt로 JS/TS 파일 포매팅
  ↓
커밋 완료 → git push
  ↓
pre-push → Git LFS 검증
```

## Context7 — LLM 문서 컨텍스트 서비스

Upstash가 만든 MCP 서버. GitHub 레포의 문서를 파싱/인덱싱하여 AI 코딩 도구(Claude Code, Cursor 등)에 최신 문서를 컨텍스트로 제공한다.

> 📍 context7.json

```json
{
  "url": "https://context7.com/tldraw/tldraw",
  "public_key": "pk_pMHtZZcPXLY8ti5lsBvjP"
}
```

`robots.txt`처럼 동작. Context7 크롤러가 레포를 인덱싱할 때 이 파일을 읽고 설정에 따라 문서를 처리. 파일만 루트에 두면 기본 rate limit으로 바로 동작하고, context7.com 대시보드에서 등록하면 더 높은 rate limit과 관리 기능 제공.

라이브러리 제공자(tldraw)가 이 파일을 넣어두면, 사용자 쪽에서 Context7 MCP 서버를 연결했을 때 tldraw API 문서를 자동으로 참조 가능.

## OxLint 린팅 설정 (ESLint + OxLint 이중 체계)

실제 `yarn lint`은 `oxlint .`을 실행한다. ESLint config(`eslint.config.mjs`)은 남아있지만 CI/스크립트에서 사용되지 않으며 레거시에 가깝다.

> 📍 .oxlintrc.json
> 📍 eslint.config.mjs
> 📍 internal/scripts/oxlint/tldraw-plugin.mjs
> 📍 관련 개념: `.study/00-dev-concepts.md` — 린팅 도구 생태계

### 커스텀 룰 플러그인

`tldraw-plugin.mjs` 하나의 JS 파일이 ESLint(`local/*`)와 OxLint(`tldraw/*`) 양쪽에서 공유된다. ESLint의 `create(context)` API로 작성되어 양쪽 모두 호환.

```js
// .oxlintrc.json — OxLint에서 JS 플러그인 로드
"jsPlugins": ["./internal/scripts/oxlint/tldraw-plugin.mjs"]

// eslint.config.mjs — ESLint에서 동일 파일 로드
import localRules from './internal/scripts/oxlint/tldraw-plugin.mjs'
plugins: { local: localRules }
```

OxLint가 ESLint의 `no-restricted-syntax` AST 셀렉터를 지원하지 않아서, ESLint에서 셀렉터로 처리하던 룰들을 별도 JS 룰로 재구현했다: `no-setter-getter`, `no-direct-storage`, `no-exported-arrow-const`, `img-referrer-policy` 등.

### 룰 분류 (~80개 활성)

**1. 크로스 윈도우 임베딩 보호** — tldraw 린팅의 핵심 차별점

`packages/editor`, `packages/tldraw`, `packages/utils`에서 브라우저 전역 직접 사용을 금지한다. tldraw가 iframe이나 다른 윈도우에 임베딩될 수 있어서, `window`/`document` 전역을 직접 쓰면 크로스 윈도우 환경에서 깨지기 때문.

| 금지 대상 | 대체 수단 |
|---|---|
| `fetch`, `Image` | `@tldraw/util` 래퍼 |
| `setTimeout`, `setInterval`, `requestAnimationFrame` | `editor.timers` |
| `document` | `editor.getContainerDocument()` |
| `getComputedStyle`, `matchMedia`, `getSelection` 등 | `getOwnerWindow()` 경유 |
| `instanceof HTMLElement` 등 | `ownerDocument.defaultView`에서 생성자 획득 |
| `structuredClone` | `@tldraw/util`의 것 |

**2. 모듈 경계 보호**
- `tldraw/no-export-star` — `export *` 금지. API 표면 통제 + 트리셰이킹
- `tldraw/no-internal-imports` — `@tldraw/editor/src/...` 같은 내부 경로 import 금지

**3. 코드 스타일 일관성**
- `tldraw/prefer-class-methods` — 클래스 내 화살표 함수 → 메서드 강제
- `tldraw/no-setter-getter` — getter/setter 금지 (reactive signal 기반이라 불필요)
- `tldraw/no-exported-arrow-const` — `export const fn = () => {}` → 함수 선언문 강제
- `tldraw/method-signature-style` — TS 인터페이스 메서드 시그니처 강제
- `consistent-type-definitions` — `type` → `interface` 강제
- `consistent-type-exports` — 인라인 타입 export 강제

**4. 국제화(i18n)** — dotcom 앱 다국어 지원
- `tldraw/enforce-default-message` — formatMessage에 defaultMessage 필수
- `tldraw/jsx-no-literals` — JSX 내 하드코딩 문자열 금지 (tla 폴더)
- `no-restricted-imports: react-intl` — 래핑된 useIntl 사용 강제

**5. React / Next.js**
- `react-hooks/rules-of-hooks`, `exhaustive-deps` — 훅 규칙 강제
- `react/jsx-no-target-blank`, `jsx-key` 등 일반적 React 린팅
- `tldraw/tagged-components` — `@public` 컴포넌트에 `@react` 태그 필수 (API 문서 생성용)

**6. 문서화 / 테스트 / 호환성**
- `tldraw/tsdoc-param-matching` — TSDoc @param과 실제 파라미터 일치 검증
- `tldraw/no-focused-tests` — `.only` 테스트 금지
- `tldraw/no-tiptap-default-import` — @tiptap default import 금지 (CJS/ESM 호환)
- `tldraw/no-whilst` — "whilst" → "while" 미국식 영어 통일

### 파일별 오버라이드

| 파일 범위 | 완화되는 룰 | 이유 |
|---|---|---|
| `**/*.test.ts` | 대부분의 제한 룰 off | 테스트에서는 전역 자유 사용 |
| `apps/examples/**` | setter/getter, storage, arrow const off | 데모 코드 단순화 |
| `templates/**` | 위 + prefer-class-methods off | 입문자용 |
| `internal/**` | no-console, no-export-star off | 내부 스크립트 |

### OxLint를 선택한 이유

- 모노레포가 거대해서 ESLint가 체감될 만큼 느림
- 14개 커스텀 룰을 JS 플러그인으로 그대로 이식 가능
- Biome는 ESLint 플러그인 호환이 안 되어 `tagged-components`, `tsdoc-param-matching` 같은 복잡한 AST 순회 룰을 GritQL로는 재구현 불가
