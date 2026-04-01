# 개발 개념 & 용어

## 빌드 vs 번들링

"빌드"는 소스 코드를 배포 가능한 결과물로 만드는 전체 과정. 번들링은 그 안의 한 단계일 수도 있고 아닐 수도 있다.

- **빌드(Build)**: TS→JS 변환, 타입 체크, 최적화 등 배포 준비의 총칭
- **번들링(Bundling)**: 여러 파일을 하나(또는 소수)로 합치는 것. 브라우저 HTTP 요청을 줄이기 위한 목적
- **트랜스파일(Transpile)**: 한 언어를 다른 언어로 변환 (TS→JS, JSX→JS 등). esbuild, SWC, Babel이 담당

SDK 라이브러리는 보통 번들링을 하지 않는다 (`bundle: false`). 파일별로 변환만 해서 내보내고, 번들링은 최종 앱 개발자의 빌드 도구에 맡긴다. 미리 번들링하면 사용자의 빌드 도구가 트리 쉐이킹(안 쓰는 코드 제거)하기 어려워지기 때문.

| | SDK (라이브러리) | 앱 (Next.js 등) |
|---|---|---|
| 번들링 | 안 함 | 함 |
| 출력 포맷 | CJS + ESM 두 벌 | 프레임워크가 결정 |
| 빌드 도구 | 직접 구성 (esbuild 등) | 프레임워크 내장 |
| 타입 정의 | `.d.ts` 직접 배포 | 불필요 |

## CJS vs ESM 듀얼 패키지

npm 패키지를 배포할 때 CommonJS와 ES Module 두 벌을 만드는 관행.

- **CJS** (`require()`): Node.js 전통 방식. `dist-cjs/index.js`
- **ESM** (`import`): 모던 표준. `dist-esm/index.mjs`

두 벌을 만드는 이유는 사용자 환경이 다양하기 때문. 오래된 Node.js 프로젝트는 CJS, 모던 프로젝트는 ESM을 쓴다.

## 모노레포 오케스트레이터

모노레포에서 여러 패키지의 빌드를 조율하는 메타 레벨 도구. 개별 빌드 도구(esbuild, tsc 등) 위에서 동작한다.

하는 일 4가지:

1. **의존성 그래프 파악** — 패키지 간 관계를 분석하여 빌드 순서 결정. A가 B에 의존하면 B를 먼저 빌드
2. **병렬 실행** — 의존 관계가 없는 패키지들은 동시에 빌드
3. **증분 빌드** — 변경된 패키지와 그에 영향받는 패키지만 빌드. 의존성 그래프를 알기 때문에 가능
4. **캐싱** — 입력 파일들의 해시를 계산하여 이전과 동일하면 빌드 건너뜀. 증분 빌드보다 한 단계 더 정밀한 판단

증분 빌드와 캐싱의 차이: 파일을 수정했다가 되돌리면, 증분 빌드는 "변경됨"으로 판단하지만 캐싱은 해시가 동일하므로 스킵한다.

대표적인 도구들:

| 도구 | 만든 곳 | 특징 |
|---|---|---|
| Turborepo | Vercel | 설정 간단, 원격 캐싱 (Vercel 클라우드 연동) |
| Nx | Nrwl | 기능 풍부, 플러그인 생태계 |
| lazyrepo | tldraw (ds300) | 경량, 자체 프로젝트에 최적화 |

단일 프로젝트에서는 빌드 도구(tsc, esbuild 등)가 자체적으로 증분 빌드를 지원하므로 오케스트레이터가 불필요. 모노레포에서 "어떤 패키지를 어떤 순서로 빌드할지"를 관리해야 할 때 필요해진다.

## 앱 빌드 vs SDK 빌드

프레임워크(Next.js 등)로 앱을 만들 때와 SDK 라이브러리를 만들 때 빌드 책임이 다르다.

**앱 개발**: 프레임워크가 빌드 파이프라인을 내장. `next build` 한 줄이면 TS→JS 변환(SWC), 번들링(Webpack/Turbopack), 코드 스플리팅, 최적화까지 전부 처리. 개발자가 빌드 스크립트를 짤 일이 없다.

**SDK 개발**: 직접 구성해야 할 것들:
- TS→JS 변환 도구 선택 및 설정 (esbuild, SWC, tsc 등)
- CJS/ESM 출력 포맷 관리
- `.d.ts` 타입 정의 생성 및 배포
- public API surface 관리
- 번들링 여부 결정

소규모 라이브러리라면 tsup 같은 라이브러리 빌드 도구로 간단히 해결 가능. tldraw처럼 패키지가 많고 요구사항이 세밀하면 `build-package.ts`처럼 직접 작성.

## 단위테스트 vs 통합테스트 vs E2E 테스트

테스트는 **격리 수준**에 따라 세 계층으로 나뉜다.

- **단위(Unit)**: 하나의 모듈을 격리해서 테스트. 외부 의존성 없이 함수/클래스 단위로 검증
- **통합(Integration)**: 여러 모듈이 **함께 동작하는 것**을 테스트. 모듈 간 접점(interface)이 올바르게 동작하는지 확인. 각 모듈이 단위테스트를 통과해도 조합하면 깨질 수 있기 때문에 필요
- **E2E(End-to-End)**: 실제 브라우저에서 전체 앱을 유저 관점으로 테스트

```
순수 단위                     통합                          E2E
←──────────────────────────────────────────────────────────────→
atom.set(2)         TestEditor로                Playwright로
expect(2)           도형 생성→선택→회전          실제 브라우저에서
                    여러 모듈 연동 확인           클릭/드래그 수행
```

핵심 구분: **"격리 vs 조합"**. 하나만 테스트하면 단위, 여러 개를 엮으면 통합.

## 테스트 시나리오 정의 방식: 코드 vs 유저 행동

단위/통합과 E2E는 **시나리오를 기술하는 언어**가 근본적으로 다르다.

**단위/통합** — 내부 코드 API로 시나리오를 정의한다:
```ts
editor.createShapes([{ type: 'geo', ... }])
editor.select(ids.box1)
editor.pointerDown(60, 10, { handle: 'top_left_rotate' })
```
프로그래머가 "이 메서드를 이 인자로 호출하면 이 결과"라고 직접 지정.

**E2E** — 유저가 실제로 하는 행동으로 시나리오를 정의한다:
```ts
await page.mouse.click(100, 100)
await page.mouse.move(200, 200)
await page.keyboard.press('r')
```
내부 API를 모름. 화면에 보이는 결과만 검증.

| | 단위/통합 | E2E |
|---|---|---|
| 질문 | "이 코드가 맞게 동작하나?" | "유저가 이걸 하면 기대한 결과가 나오나?" |
| 시나리오 언어 | 코드 API | 유저 행동 (클릭, 타이핑, 드래그) |
| 검증 대상 | 내부 상태 | 화면 결과 (스크린샷, DOM 요소) |
| 깨지는 이유 | 로직 버그 | 로직 + 렌더링 + 이벤트 바인딩 버그 |

통합테스트의 `TestEditor.pointerDown()`은 유저 행동을 코드로 흉내낸 것이고, E2E의 `page.mouse.click()`은 실제 브라우저에서 진짜 이벤트를 발생시키는 것. 그 사이에 이벤트 바인딩, DOM 렌더링, CSS 레이아웃 같은 레이어가 포함되느냐가 차이.

## TSDoc — TypeScript 주석 표준 규격

TypeScript 코드의 JSDoc 주석을 표준화한 규격. Microsoft가 만들었다.

JSDoc은 자유 형식이라 도구마다 해석이 달랐다. TSDoc은 파싱 규칙을 통일하여 모든 도구(API Extractor, IDE, 문서 생성기 등)가 동일하게 해석한다.

**JSDoc과의 핵심 차이**: TypeScript가 이미 타입 정보를 갖고 있으므로, `{string}` 같은 타입 표기를 주석에 쓰지 않는다. 대신 릴리스 태그와 문서 구조에 집중한다.

```ts
// JSDoc (자유 형식)
/** @param {string} name - 사용자 이름 */

// TSDoc (표준화)
/** @param name - 사용자 이름
 *  @public */
```

TSDoc 자체는 파서 규격일 뿐이다. 이를 소비하는 도구가 실제 가치를 만든다:

| 도구 | TSDoc을 어떻게 쓰는가 |
|------|----------------------|
| API Extractor | `@public`/`@internal` 태그로 공개 API 추출 및 분류 |
| API Documenter | 주석 내용으로 API 문서 자동 생성 |
| IDE (VSCode 등) | 호버 시 주석을 툴팁으로 표시 |

### 릴리스 태그 체계

| 태그 | 의미 |
|------|------|
| `@public` | 공개 API — SDK 사용자에게 노출 |
| `@internal` | 내부 전용 — public.d.ts에서 제외 |
| `@beta` | 베타 API |
| `@alpha` | 알파 API |

태그를 안 붙이면 API Extractor는 기본적으로 `@public`으로 간주한다. 즉 태그는 "숨기고 싶을 때" 필요한 것.

### 커스텀 태그

`tsdoc.json`에서 프로젝트별 커스텀 태그를 정의할 수 있다. tldraw는 `@react` modifier를 정의하여 React 컴포넌트/훅임을 표시한다.

> 📍 관련 상세: `.study/05-api-design.md` — TSDoc + API Extractor 파이프라인

## API Extractor — d.ts 롤업 + 릴리스 태그 분류

Microsoft의 도구로, 패키지의 흩어진 `.d.ts` 파일들을 하나로 합치고(rollup), TSDoc 릴리스 태그에 따라 공개/비공개를 분리한다.

하는 일은 크게 두 가지:

1. **d.ts 롤업**: 수십 개의 선언 파일을 진입점(`index.d.ts`)에서 export 체인을 따라가며 단일 파일로 병합
2. **릴리스 태그 필터링**: `@public`만 포함한 `public.d.ts`와 전체를 포함한 `internal.d.ts`를 각각 생성

도구 자체의 기능은 단순하지만, 이 산출물을 빌드 파이프라인에 엮어 CI 검증, API 변경 추적, 문서 자동 생성을 자동화하는 구조가 핵심.

여기서 말하는 "API"는 서버 REST API가 아니라, **패키지가 외부에 노출하는 모든 public 인터페이스** — 클래스, 함수, 타입, 상수 등 `import { ... } from '@tldraw/editor'`로 가져다 쓸 수 있는 모든 export를 뜻한다.

> 📍 관련 상세: `.study/05-api-design.md` — TSDoc + API Extractor 파이프라인

## npm 배포 (Publishing)

`npm publish` 한 명령으로 패키지를 npm 레지스트리에 업로드한다. GitHub 레포, 심사/승인 과정 불필요. npm 계정(무료)과 `package.json`만 있으면 즉시 전 세계에서 설치 가능.

### 최소 요구사항

```json
{ "name": "my-package", "version": "1.0.0" }
```

이것만 있어도 배포 가능. 실무에서는 `main`(진입점), `files`(포함 파일), `.npmignore`(제외 파일)를 추가.

### Semantic Versioning (SemVer) & 버전 Bump

같은 버전을 두 번 올릴 수 없다. 반드시 버전을 올려야 함. **"version bump"**이란 이 버전 숫자를 한 단계 올리는 것을 말한다.

```
3.12.1
│  │  └─ patch — 버그 수정
│  └──── minor — 새 기능 추가 (하위 호환)
└─────── major — 호환성 깨지는 변경
```

| bump 타입 | 예시 | 언제 |
|-----------|------|------|
| patch bump | `3.12.1` → `3.12.2` | 버그 수정 |
| minor bump | `3.12.1` → `3.13.0` | 새 기능 (patch는 0으로 리셋) |
| major bump | `3.12.1` → `4.0.0` | 호환 깨지는 변경 (minor, patch 리셋) |

```bash
npm version patch   # 4.5.3 → 4.5.4
npm version minor   # 4.5.3 → 4.6.0
npm version major   # 4.5.3 → 5.0.0
npm publish
```

`npm version`은 package.json version 수정 + git commit + git tag 생성.

모노레포에서는 패키지가 수십 개이므로, 개별 `npm version`이 아니라 커스텀 스크립트로 **모든 package.json을 한꺼번에** bump한다.

> 📍 tldraw 적용: `.study/01-dev-infra.md` — 퍼블리시 파이프라인 상세

### dist tag

배포 트랙을 분리하는 메커니즘. 기본은 `latest`.

```bash
npm publish                 # → latest (안정 버전)
npm publish --tag canary    # → canary (프리릴리스)
npm publish --tag next      # → next (릴리스 후보)
```

사용자가 `npm install foo`하면 `latest` 태그 버전이 설치됨. 다른 트랙은 `npm install foo@canary`로 명시.

### Private 배포

| 방법 | 설명 |
|---|---|
| npm Organizations | 유료 (월 $7/user), `"access": "restricted"` 설정 |
| GitHub Packages | GitHub 레포 권한 기반, Enterprise에 포함 |
| Verdaccio | 오픈소스 셀프 호스팅 레지스트리 |
| AWS CodeArtifact / Google Artifact Registry | 클라우드 매니지드 |

GitHub Enterprise 사용 시 GitHub Packages가 가장 간단 — 별도 서버 없이 `.npmrc`에 레지스트리 지정만 하면 됨.

```
@your-org:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

`"private": true`를 package.json에 설정하면 `npm publish` 자체가 차단됨 — 모노레포 루트처럼 배포 대상이 아닌 패키지의 실수 방지용.

## Lerna — 모노레포 npm 패키지 버전/배포 도구

npm에 배포할 패키지가 **여러 개이고 서로 의존**할 때 필요한 도구. 단일 패키지거나 의존성이 없으면 불필요.

### 해결하는 문제

| 문제 | Lerna 없이 | Lerna 있으면 |
|---|---|---|
| 버전 올리기 | N개 package.json 수동 수정 | 한 명령으로 일괄 변경 |
| npm 퍼블리시 | 의존성 순서 직접 파악해서 하나씩 | 토폴로지컬 정렬로 자동 순서 퍼블리시 |
| 변경 감지 | git diff 보면서 직접 판단 | 변경된 패키지 자동 감지 |

### 버전 모드

| 모드 | 설명 |
|---|---|
| **Fixed (고정)** | 모든 패키지가 같은 버전 (예: 전부 4.5.3) |
| **Independent (독립)** | 패키지마다 다른 버전 가능 |

### 퍼블리시 순서가 중요한 이유

```
❌ @tldraw/tldraw 먼저 publish → @tldraw/editor@4.5.4에 의존
   → @tldraw/editor@4.5.4는 아직 npm에 없음 → 설치 실패

✅ @tldraw/state → @tldraw/store → @tldraw/editor → @tldraw/tldraw
   (의존성 없는 것부터 있는 것 순으로)
```

## 모듈 분리 전략 — 단계적 접근

코드 분리는 점진적으로 진행하는 것이 원칙. 너무 이른 npm 분리는 오히려 생산성을 떨어뜨린다.

```
1단계: 폴더 분리
src/utils/, src/auth/, src/api/

2단계: 모노레포 패키지 분리
packages/utils/, packages/auth/, packages/api/
→ 같은 레포, 배포 없이 바로 참조, 디버깅 편리

3단계: npm 패키지 배포
→ 다른 프로젝트에서도 npm install로 사용
```

### npm 분리의 비용

- 버전 관리 오버헤드 발생
- 변경 → 배포 → 설치 사이클로 개발 속도 저하
- 디버깅이 어려워짐 (node_modules 안에 있으므로)

**모노레포 패키지로 충분한데 npm으로 분리하면 손해.** tldraw도 내부 개발은 모노레포로 하고, 외부 사용자 제공 목적으로만 npm에 배포한다.

> 📍 관련 상세: `.study/01-dev-infra.md` — tldraw의 Lerna + 커스텀 퍼블리싱 인프라
> 📍 관련 상세: `.study/03-architecture-patterns.md` — 모듈 분리 패턴

## `.ignore` 파일 — 검색 도구용 제외 규칙

`.gitignore`와 문법은 같지만, **ripgrep(`rg`), The Silver Searcher(`ag`), `fd`** 같은 코드 검색/파일 탐색 도구가 읽는 별도 제외 규칙 파일.

| | `.gitignore` | `.ignore` |
|---|---|---|
| **읽는 주체** | Git | ripgrep, ag, fd 등 검색 도구 |
| **목적** | Git 추적에서 제외 | 검색 결과에서 제외 |
| **영향** | 버전 관리 | 코드 검색 시 노이즈 제거 |

**왜 별도로 필요한가**: `.gitignore`에 이미 빌드 파일이 제외되어 있지만, Git이 추적하는 파일 중에서도 검색 시 불필요한 것들이 있다. `yarn.lock`(수천 줄), `*.d.ts`(타입 선언), 자동 생성 문서 등은 커밋은 되지만 코드 검색에는 노이즈.

**VS Code 검색에도 적용**: VS Code의 `Cmd+Shift+F` 파일 검색이 내부적으로 ripgrep을 사용하므로, 프로젝트 루트에 `.ignore`를 두면 팀 전체가 동일한 검색 필터를 자동 공유.

## Docker 빌드 컨텍스트

`docker build .` 실행 시 Docker는 **파일시스템 기준**으로 현재 디렉토리 전체를 Docker 데몬에 전송한다. Git 추적 여부와 무관하게 디스크에 존재하는 모든 파일이 대상.

`.dockerignore`는 `.gitignore`처럼 "추적 제외"가 아니라, **"디렉토리에서 Docker 데몬으로 복사할 때 빼라"**는 파일시스템 레벨 필터.

### `.git` 디렉토리와 `.dockerignore`

`.git`은 Git의 내부 디렉토리로, Git이 추적하는 파일이 아니다 (커밋에 포함되지 않음). 하지만 로컬 디스크에는 존재하므로, `docker build .`를 로컬에서 실행하면 Docker 데몬에 같이 전송된다. `.git`은 수백 MB~수 GB가 될 수 있어서 `.dockerignore`에 넣어 전송량을 줄인다.

**단, CI/CD 환경(쿠버네티스 등)에서는**: `git clone`으로 받아온 소스에 `.git`이 없거나 shallow clone이라 작아서 `.dockerignore`의 실질적 필요성이 낮다. `.dockerignore`가 주로 의미 있는 건 **로컬에서 `docker build`하는 개발자** 케이스.

### 현실적으로 `.dockerignore`에서 의미 있는 항목

- **`.git`** — 로컬 빌드 시 용량이 크니까
- **`node_modules`** — 컨테이너 안에서 새로 설치하니까

나머지(`README.md`, `LICENSE` 등 문서류)는 수 KB 수준이라 넣어도 안 넣어도 실질적 차이 없음.

## CI (Continuous Integration) — 코드 품질 자동 검증

PR이나 push 시 자동으로 코드 품질을 검증하는 파이프라인. 보통 GitHub Actions 같은 CI 서비스에서 실행.

### 일반적인 CI 체크 구성

비용이 싼 체크를 먼저, 비싼 체크를 나중에 돌려서 실패 시 빠르게 끊는다.

| 카테고리 | 체크 예시 | 일반적인가 |
|---|---|---|
| **의존성 검증** | constraints, dedupe | 모노레포에서 흔함 |
| **패키지 구조** | circular-deps, check-packages | 모노레포에서 흔함 |
| **정적 분석** | typecheck, lint | 거의 모든 프로젝트 기본 |
| **API 계약** | api-check | SDK/라이브러리 프로젝트 |
| **생성물 검증** | bundle-size, i18n diff | SDK/다국어 앱 |
| **테스트** | unit test | 거의 모든 프로젝트 기본 |
| **빌드 검증** | build | 거의 모든 프로젝트 기본 |

전체 흐름: **의존성 OK → 코드 OK → 산출물 OK → 동작 OK**

### CI에서의 빌드

CI에서 빌드를 돌리는 목적은 **배포가 아니라 검증**. "이 코드가 빌드가 되긴 하는가?"를 확인하여, 빌드가 깨지는 코드가 merge되는 걸 방지한다. 실제 배포는 별도 워크플로우에서 수행.

## Lock 파일 구조 — 요구사항 vs 실제 설치 버전

yarn.lock(또는 package-lock.json)은 package.json의 **범위 지정자**(semver range)를 **정확한 버전**으로 고정(lock)한 파일이다.

```
lodash@^4.17.0:                  ← 요구사항 (package.json의 "lodash": "^4.17.0")
  version: 4.17.20               ← 실제 설치된 정확한 버전
  resolved: "https://registry..."  ← 다운로드한 파일의 정확한 URL
```

- `^4.17.0`은 "4.17.0 이상, 5.0.0 미만 아무거나"라는 뜻
- 오늘 설치하면 4.17.23, 한 달 전에 설치하면 4.17.21이 될 수 있음
- lock 파일이 "이 요구사항에 대해 실제로 이 버전을 설치했다"를 고정해둠

**요구사항 범위가 다르면 별도 엔트리로 관리됨:**

```
# 6개월 전에 editor 패키지 설치 → 그때 최신이 4.17.20
lodash@^4.17.0:
  version: 4.17.20

# 3개월 전에 tldraw 패키지 추가 → 그때 최신이 4.17.21
lodash@^4.17.20:
  version: 4.17.21
```

`^4.17.0`도 4.17.21을 쓸 수 있지만, lock 파일에 이미 4.17.20으로 고정되어 있으니 그대로 유지됨.

## Dependabot — GitHub 자동 의존성 업데이트 봇

GitHub이 기본 제공하는 봇으로, 프로젝트 의존성의 새 버전이 나오면 자동으로 업데이트 PR을 만들어준다.

**두 가지 모드:**

| 구분 | Dependabot version updates | Dependabot security alerts |
|---|---|---|
| 설정 | `.github/dependabot.yml` 필요 | 설정 없이 기본 활성화 |
| 트리거 | 정기적(daily/weekly 등) | 보안 취약점 발견 시에만 |
| 목적 | 최신 버전 유지 | 보안 패치 |

**PR 자동 생성 예시:**

```
브랜치: dependabot/npm_and_yarn/esbuild-0.25.6
제목: Bump esbuild from 0.25.5 to 0.25.6

본문:
- 변경 내역 표 (From → To)
- Release notes (접힌 상태)
- Changelog (접힌 상태)
- Commits (접힌 상태)
- @dependabot 명령어 안내 (rebase, merge, close 등)
```

**Dependabot의 동작 방식은 "최소한의 변경만":** 자기가 업데이트하는 패키지의 lock 파일 엔트리만 수정하고, 다른 패키지들은 건드리지 않음. lock 파일 전체를 최적화하는 건 자기 역할이 아님.

> 📍 tldraw 적용: `.github/dependabot.yml`은 2024년에 삭제됨 (version updates 비활성화). 보안 alerts만 활성화된 상태.

## yarn dedupe — 의존성 중복 제거

yarn.lock에 같은 패키지가 다른 버전으로 중복 설치된 경우를 정리한다.

```
# 중복 상태 (dedupe 필요)
lodash@^4.0.0 → 4.17.20
lodash@^4.1.0 → 4.17.21    ← 둘 다 4.17.21로 통일 가능

# dedupe 후
lodash@^4.0.0 → 4.17.21
lodash@^4.1.0 → 4.17.21    ← 하나로 통일
```

**문제점**: 번들 크기 증가(같은 라이브러리 2벌), 런타임 버그(React 2개 로드 시 hooks 에러), 설치 시간 증가.

**원인**: `yarn add` 시 그 시점의 최신 버전으로 resolve되므로, 시간이 지나면서 같은 패키지의 다른 버전들이 쌓임. yarn이 자동으로 통일해주지는 않음.

- `yarn dedupe` — yarn.lock을 실제 수정하여 중복 제거
- `yarn dedupe --check` — 수정 없이 중복 여부만 확인 (CI용)

### Dependabot이 중복을 악화시키는 패턴

Dependabot은 하나의 패키지만 업데이트하므로, 기존에 통합되어 있던 lock 파일에 오히려 중복이 생길 수 있다:

```
# 현재 상태 (잘 통합되어 있음)
lodash@^4.17.0 → 4.17.21
lodash@^4.17.20 → 4.17.21

# Dependabot이 한 패키지에서 lodash를 ^4.17.23으로 올림
lodash@^4.17.0 → 4.17.21    ← 안 건드림
lodash@^4.17.20 → 4.17.21   ← 안 건드림
lodash@^4.17.23 → 4.17.23   ← 새로 추가 (자기가 바꾼 것만)

# yarn dedupe 실행 후
lodash@^4.17.0, lodash@^4.17.20, lodash@^4.17.23 → 4.17.23
```

이 문제를 자동으로 해결하기 위해 tldraw는 `dependabot-dedupe` 워크플로우를 운영한다.

> 📍 상세: `.study/01-dev-infra.md` — dependabot-dedupe 워크플로우

## CI 러너와 캐싱 전략

### CI 러너는 Ephemeral(일회용) VM

GitHub Actions 러너는 **매번 새 VM**이 뜨고 Job 끝나면 사라진다. 메모리나 디스크에 뭘 남길 수 없다.

**용어 구분**: "Edge Runtime"은 Cloudflare Workers, Vercel Edge Functions 같은 경량 V8 isolate 환경을 가리키는 별도 개념. GitHub Actions 러너는 완전한 VM(Ubuntu, root 권한, Docker 실행 가능)이므로 Edge Runtime이 아니다. 정확한 표현은 **ephemeral**(일회용) 또는 **stateless**(상태 비유지).

### 대형 러너 (Larger Runners)

GitHub 기본 러너는 2코어 VM이지만, 유료로 **대형 러너**를 쓸 수 있다.

| `runs-on` 값 | 스펙 | 비용 |
|---|---|---|
| `ubuntu-latest` | 2코어, 기본 제공 | 무료 (공개 레포) |
| `ubuntu-latest-16-cores-arm-open` | 16코어 ARM | 유료 |

대형 러너에도 Node, Docker, git, gh CLI 등 개발 도구가 사전 설치되어 있다. 모노레포 빌드 + npm 퍼블리시 같은 무거운 작업에서 사용.

> 📍 tldraw 적용: 퍼블리시 워크플로우 대부분이 16코어 ARM 러너 사용

### 캐싱은 외부 스토리지

`actions/cache`가 GitHub의 별도 스토리지(S3 같은)에 캐시를 저장/복원한다.

```
Job 시작 → GitHub 스토리지에서 캐시 tar.gz 다운로드 → 압축 해제 → 작업 수행
Job 끝 → 캐시 파일들을 tar.gz로 압축 → GitHub 스토리지에 업로드
```

매 Job마다 다운로드/업로드 오버헤드가 있지만, 전체를 처음부터 빌드하는 것보다 훨씬 빠르다.

- 캐시 용량 제한: 레포당 10GB, 넘으면 오래된 것부터 삭제
- 7일간 미사용 시 자동 삭제
- branch별 캐시 분리 (PR 브랜치는 base branch 캐시도 참조 가능)

### 2단계 fallback 캐시 키

```yaml
key: tools-${{ runner.os }}-${{ hashFiles('yarn.lock') }}-${{ github.sha }}
restore-keys: |
  tools-${{ runner.os }}-${{ hashFiles('yarn.lock') }}-
```

1. **정확한 매칭**: OS + 의존성 해시 + 커밋 SHA가 완전히 같으면 → 캐시 히트
2. **prefix 매칭** (fallback): SHA 없이 OS + 의존성 해시만 일치 → 이전 커밋의 캐시 복원

yarn.lock이 바뀌면 해시가 달라지므로 캐시가 무효화되고 처음부터 다시 빌드. 의존성이 바뀌었으니 당연히 그래야 함.

### 개인 프로젝트에서의 캐싱

`actions/cache`는 GitHub 공식 액션이라 그냥 가져다 쓰면 된다.

```yaml
- uses: actions/cache@v5
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

소규모 프로젝트면 `node_modules` 캐싱만으로도 충분.

## `dorny/paths-filter` — CI에서 경로 기반 조건부 실행

PR이나 push에서 **변경된 파일 경로를 검사**하여, 관련 없는 CI 워크플로우를 스킵하는 GitHub Action.

### 동작 원리

1. git diff에서 변경된 파일 목록을 가져옴
2. 워크플로우에 정의된 필터 패턴(glob)과 대조
3. 매칭 결과(`true`/`false`)를 job output으로 내보냄
4. 이후 job이 `if: needs.check.outputs.relevant == 'true'`로 실행 여부 결정

```yaml
check:
  outputs:
    relevant: ${{ steps.filter.outputs.relevant }}
  steps:
    - uses: dorny/paths-filter@v3
      with:
        filters: |
          relevant:
            - 'packages/**'        # 포함 패턴
            - 'apps/dotcom/**'
            - '!**/*.md'           # 제외 패턴 (! prefix)

build:
  needs: check
  if: needs.check.outputs.relevant == 'true'  # false면 job 자체가 생성 안 됨
```

### 포함/제외 패턴

- **포함** (`!` 없음): 이 경로에 변경이 있으면 `true`
- **제외** (`!` prefix): 포함 패턴에 매칭되더라도 이 패턴**만** 변경되면 `false`

예: `packages/editor/README.md`만 수정 → `packages/**`에 매칭되지만 `!**/*.md`로 제외 → `false`

### 핵심 포인트

- 자동으로 의존성을 분석해주는 것이 아님. **각 워크플로우 작성자가 관련 경로를 수동으로 지정**
- `check` job은 `ubuntu-latest`(2코어 무료)에서 수 초 만에 끝나므로 게이트 비용이 거의 없음
- 모노레포에서 필수적 — 문서 오타 하나 고친 PR에도 32코어 러너가 20분 돌아가는 낭비 방지

## corepack — 패키지 매니저 버전 고정

Node.js에 내장된 패키지 매니저(yarn, pnpm) 버전 관리 도구. `package.json`의 `packageManager` 필드를 읽어 **자동으로 해당 버전을 사용**하게 한다.

```json
"packageManager": "yarn@4.9.1"
```

nvm이 Node 버전을 고정하듯, corepack은 yarn/pnpm 버전을 고정한다. 수동 설치 없이 프로젝트마다 정확한 버전이 사용됨을 보장.

## 린팅(Linting) 도구 생태계: ESLint vs OxLint vs Biome

JS/TS 린터는 "ESLint가 느리다"는 문제에서 Rust 기반 대안들이 등장하면서 세 갈래로 나뉘었다.

### ESLint — 가장 유연, 가장 느림

- JS/TS로 룰 작성, npm 패키지로 배포. 생태계 수천 개 플러그인
- AST 비지터 패턴 (`create(context)` API)으로 커스텀 룰 작성
- `no-restricted-syntax`로 AST 셀렉터만으로도 룰 생성 가능 (코드 없이)
- 타입 정보 접근 가능 (`@typescript-eslint/parser`의 TypeScript 서비스 연동)
- Flat Config (`eslint.config.mjs`) — 새로운 설정 형식. extends 대신 배열로 구성

```json
// AST 셀렉터 예시 — 코드 없이 룰 생성
"no-restricted-syntax": ["error", {
  "selector": "MethodDefinition[kind='set']",
  "message": "Property setters are not allowed"
}]
```

### OxLint (Oxc 프로젝트) — ESLint 호환 + Rust 속도

- Oxc = Oxidation Compiler. Rust로 JS/TS 툴체인 전체를 다시 만드는 프로젝트 (파서, 린터, 포매터, 리졸버, 트랜스파일러)
- 내장 룰은 Rust로 작성 → 빠름
- **JS 플러그인(`jsPlugins`)** 지원 — ESLint의 `create(context)` API 호환. 기존 ESLint 커스텀 룰을 그대로 이식 가능
- `typeAware: true`로 타입 정보 활용 가능
- 한계: `no-restricted-syntax` 같은 AST 셀렉터 미지원 → 별도 JS 룰로 재구현 필요. JS 플러그인은 JS 런타임에서 돌아서 해당 룰만큼은 ESLint 속도와 비슷

### Biome (구 Rome) — 통합 도구, 확장성 제한

- 린터 + 포매터를 **하나의 통합 도구**로 제공. Prettier + ESLint를 합친 느낌
- 자체 룰 체계 — ESLint 플러그인 호환 안 됨
- 커스텀 룰 = GritQL (자체 패턴 매칭 언어) 또는 Rust 플러그인
- 설정이 단순하고 zero-config에 가까움
- 서드파티 플러그인 생태계 거의 없음

### 비교 요약

| | ESLint | OxLint | Biome |
|---|---|---|---|
| **커스텀 룰 언어** | JS/TS | JS (호환) + Rust (내장) | GritQL + Rust |
| **AST 셀렉터** | `no-restricted-syntax` | 미지원 | GritQL로 유사하게 |
| **타입 정보 활용** | 가능 | 가능 | 제한적 |
| **ESLint 플러그인 재활용** | 당연히 됨 | JS 플러그인으로 됨 | 안 됨 |
| **서드파티 생태계** | 수천 개 | ESLint 것 일부 호환 | 거의 없음 |
| **속도** | 느림 | 빠름 (JS 룰은 느림) | 빠름 |
| **포매터 포함** | 아니오 (Prettier 별도) | oxfmt (별도) | 내장 |

### 선택 기준

- **커스텀 룰이 복잡하고 많다** → OxLint (JS 플러그인 호환)
- **기존 ESLint 생태계 유지** → ESLint (속도가 문제 아니면)
- **설정 단순, 커스텀 룰 불필요** → Biome (린터+포매터 통합)

## PR 프리뷰 배포 (Preview Deployment)

PR의 코드 변경사항을 실제 동작하는 임시 URL로 배포해서, merge 전에 브라우저에서 직접 확인할 수 있게 하는 것.

Vercel, Netlify, Cloudflare Pages 등 모던 배포 플랫폼이 기본 기능으로 제공한다. GitHub 레포를 연결하면 PR마다 자동으로 프리뷰 URL이 생성됨.

단, 프론트엔드만 프리뷰되므로, 백엔드 서비스(DB, Worker 등)가 필요한 풀스택 앱은 플랫폼 기본 프리뷰만으로 부족하다. 이 경우 CI에서 직접 백엔드까지 포함한 풀스택 프리뷰 환경을 구축해야 한다.

> 📍 tldraw 사례: `.study/01-dev-infra.md` — deploy-dotcom.yml 워크플로우

## Dry Run — 실행 없이 검증만

배포/인프라 도구에서 **실제 실행 없이 시뮬레이션만 돌려보는 것**. "실탄 없이 쏴보는" 군사 용어에서 유래.

실제로는 빌드까지만 수행하고 업로드/적용은 하지 않는다. 설정 오류, 시크릿 누락, 문법 에러 등을 사전에 잡을 수 있다.

여러 서비스를 배포할 때 **전부 dry run 통과 → 실배포** 순서로 하면, 일부만 배포되는 반쪽짜리 상태를 방지할 수 있다. DB 트랜잭션의 all-or-nothing과 같은 발상.

```bash
# 대부분의 배포 도구가 지원하는 표준 기능
wrangler deploy --dry-run    # Cloudflare
terraform plan               # Terraform
kubectl apply --dry-run=server  # Kubernetes
cdk diff                     # AWS CDK
```

## 로컬-퍼스트 (Local-First)

데이터를 로컬에 먼저 저장하고 즉시 UI에 반영한 뒤, 백그라운드에서 서버와 동기화하는 아키텍처.

일반 웹앱은 `사용자 액션 → 서버 요청 → 응답 → UI 업데이트`로 네트워크 지연만큼 기다리지만, 로컬-퍼스트는 `사용자 액션 → 로컬 즉시 반영 → 백그라운드 서버 동기화`로 즉각 반응한다.

개발자 입장에서는 하나의 파사드로 데이터를 읽고 쓸 뿐이고, 로컬 저장/서버 동기화/충돌 해결은 엔진이 알아서 처리한다.

단순 debounce(마지막 값만 전송)가 아니라 **모든 mutation을 순서 보장하여 배치 전송**하고, 서버 거부 시 로컬 롤백(optimistic → revert), 부분 동기화(필요한 데이터만 구독) 등 분산 시스템의 어려운 문제를 처리한다.

무한 캔버스, 실시간 협업 에디터처럼 초당 수십 번 상태가 바뀌고 오프라인 지원이 필요한 앱에 적합.

> 📍 tldraw 사례: `.study/11-multiplayer-sync.md` — Zero 백엔드

## 에셋 병합 (Asset Coalescing)

SPA 배포 시 이전 빌드의 에셋(JS 청크, CSS, 이미지)을 새 배포에 함께 포함시켜, 기존 사용자의 구 버전 파일 요청이 404가 되지 않게 하는 기법.

Vite/Webpack 빌드는 파일명에 해시를 포함(`index-a3f2b1.js`)하므로 매 배포마다 파일명이 바뀐다. 이미 앱을 열어둔 사용자의 브라우저는 구 파일명으로 lazy-load 청크를 요청할 수 있다.

일반 웹앱은 CDN 캐시로 충분하지만, **장시간 세션 SPA**(무한 캔버스, 에디터, 문서 도구)는 탭을 며칠씩 열어두므로 CDN 캐시 만료 후에도 구 에셋이 필요하다. 이 경우 오브젝트 스토리지에 에셋을 백업하고 매 배포 시 병합하는 방식을 직접 구현해야 한다.

대부분의 앱에선 배포 플랫폼(Vercel, Netlify 등)이 알아서 처리해주므로 신경 쓸 필요 없다.

> 📍 tldraw 사례: `.study/09-resource-management.md` — R2 에셋 병합

## CRDT (Conflict-free Replicated Data Type)

여러 노드가 동시에 데이터를 수정해도 최종적으로 같은 상태로 수렴하는 것이 수학적으로 보장되는 자료구조.

- 서버 없이 P2P로도 동작 가능
- 어떤 순서로 변경이 도착해도 결과가 동일 (교환법칙, 결합법칙 성립)
- 별도 충돌 해결 로직 불필요 — 자료구조 자체가 해결
- 대표 라이브러리: **Yjs**, **Automerge**

### 실시간 협업의 주요 접근법

| 접근법 | 대표 사례 | 특징 |
|--------|-----------|------|
| OT (Operational Transform) | Google Docs | 서버가 연산을 변환하여 순서 보장 |
| CRDT | Figma(부분), Yjs, Automerge | 서버 없이도 수렴 보장 |
| 서버 권위 + diff 동기화 | tldraw | 서버가 최종 판정, CRDT 아이디어 차용 |
| 폴링/마지막 쓴 사람 승리 | 단순 협업 도구 | 가장 단순하지만 충돌에 취약 |

tldraw는 순수 CRDT를 쓰지 않고 서버 권위(server-authoritative) 모델을 택했다. 레코드 단위 Put/Patch/Delete 연산이 부분적으로 교환법칙을 만족하지만, 최종 판정을 서버가 하므로 엄밀히는 CRDT가 아니다.

> 📍 tldraw 적용 상세: `.study/11-multiplayer-sync.md` — tlsync 동기화 프로토콜

## 서버리스 vs Durable Objects vs 전통 서버

서버리스는 요청 처리 후 즉시 소멸하는 무상태 모델. Durable Objects는 그 한계를 보완한 상태 유지 서버리스.

| | 전통 서버 | Durable Objects | 서버리스 (Lambda, Workers) |
|---|---|---|---|
| 수명 | 영구 (직접 종료 전까지) | 연결 있으면 유지, 유휴 시 동면 | 요청 처리 후 소멸 |
| 상태 | 메모리 유지 | 메모리 유지 + 동면 시 보존 | 없음 |
| 비용 | 상시 과금 | 활성 시만 과금 | 호출당 과금 |
| WebSocket | 가능 | 가능 (Hibernation 지원) | 사실상 불가 |
| 스케일링 | 수동 | 자동 (ID당 1인스턴스) | 자동 |

### Durable Objects 핵심 특성

Cloudflare가 만든 **상태 유지 가능한 서버리스 인스턴스**. 일반 서버리스의 무상태 문제를 해결한다.

1. **전 세계에 단 하나의 인스턴스** — 같은 ID로 요청하면 어디서든 동일한 인스턴스
2. **메모리 내 상태 유지** — 변수가 다음 요청에서도 살아있음
3. **WebSocket을 직접 소유** — 여러 연결을 메모리에서 관리
4. **내장 저장소** — 인스턴스마다 전용 SQLite/KV 스토리지

### 라이프사이클

```
[Cold Start] → constructor(), 상태 로드
     ↓
[Active] → 메시지 처리, WebSocket 유지
     ↓ (모든 연결 끊김 + 유휴)
[Hibernation] → 메모리 해제, WebSocket은 플랫폼이 보관
     ↓ (메시지 도착 시 다시 깨움)
[Evicted] → 오래 방치 시 완전 소멸, 다음 요청 시 Cold Start
```

Hibernation 중에도 클라이언트의 WebSocket 연결은 끊기지 않는다. 메시지가 오면 DO를 깨워서 핸들러를 호출한다.

### Redis와의 비교

Redis로 멀티플레이어를 구현할 수 있지만, Redis는 공유 저장소이지 연결 관리나 로직 실행을 하지 않는다. 여전히 WebSocket을 유지할 서버 프로세스가 별도로 필요하고, 서버가 여러 대면 Pub/Sub 브로드캐스트 레이어를 직접 구현해야 한다. DO는 방당 인스턴스가 하나라서 이 문제가 없다.

> 📍 tldraw 적용 상세: `.study/11-multiplayer-sync.md` — Cloudflare 아키텍처

## Git Plumbing vs Porcelain — 저수준 명령어로 커밋 직접 조립

Git 명령어는 두 계층으로 나뉜다.

```
Porcelain (고수준)  ← 우리가 평소 쓰는 것
  git commit, git merge, git log, git pull ...

Plumbing (저수준)   ← Git 내부 동작을 직접 조작
  git commit-tree, git update-ref, git hash-object, git cat-file ...
```

`git merge`를 실행하면 내부적으로 `commit-tree`, `update-ref` 같은 plumbing 명령어들이 순차 실행된다.

### `git commit-tree`

Git 커밋은 세 가지 데이터의 조합이다:

```
커밋 = 트리(파일 스냅샷) + 부모(들) + 메타데이터(메시지, 작성자, 시간)
```

`git commit-tree`는 이 세 가지를 직접 지정해서 커밋 객체를 수동으로 조립한다.

```bash
commit=$(git commit-tree \
  "$TREE_HASH" \          # 어떤 파일 내용을 쓸지 (트리 객체)
  -p "$parent1" \         # 부모 커밋 1
  -p "$parent2" \         # 부모 커밋 2
  -m "커밋 메시지")        # 메시지
# → 새 커밋의 SHA 반환

git update-ref refs/heads/branch "$commit"  # 브랜치가 이 커밋을 가리키게 함
```

`git commit`이 "현재 스테이징된 파일로 커밋을 만들어줘"라면, `git commit-tree`는 "내가 재료를 다 줄 테니 커밋 하나 만들어"이다.

### `git merge`와의 차이

| | `git merge` | `git commit-tree` |
|---|---|---|
| 충돌 처리 | 자동 시도, 실패 시 사용자 개입 | 없음 (트리를 직접 지정) |
| 결과 트리 | 두 브랜치를 **병합** | 지정한 트리를 **그대로** 사용 |
| working directory | 변경됨 | 건드리지 않음 |

주로 CI/CD 자동화에서 머지 충돌 없이 항상 안전하게 브랜치를 업데이트해야 할 때 사용한다. `git merge -s theirs`(상대방 것을 100% 채택)와 비슷한 효과인데, Git에 기본 제공되지 않아서 `commit-tree`로 직접 구현하는 것이다.

> 📍 tldraw 적용: `.study/01-dev-infra.md` — `trigger-production-build.yml`

## 릴리스 트레인 (Release Train) — 배포 브랜치 전략

코드 배포를 위한 브랜치 전략은 여러 가지가 있다.

### 주요 전략 비교

**1) production 브랜치 분리** — main과 별도의 production 브랜치를 두고, 수동 트리거로 배포
```
main (개발) → staging 자동 배포
production (배포) → 수동 트리거로 프로덕션 배포
```

**2) Git Flow** — develop, release, main, hotfix 브랜치를 역할별로 운영
```
develop → release/x.x → main (= production)
hotfix → main + develop에 머지
```

**3) Trunk-based (GitHub Flow)** — 브랜치 하나로 단순하게
```
main에 직접 머지 → 바로 배포
```

**4) 태그 기반** — 특정 커밋에 버전 태그를 달아 배포
```
main에서 v1.2.3 태그 → 태그 기준으로 배포
```

### Hotfix 전략

production 브랜치 분리 방식에서 hotfix는 main을 거치지 않고 바로 production으로 배포한다. 이후 main에 반영하는 건 자동화하지 않고 개발자가 수동으로 처리한다:

- **cherry-pick** — hotfix 커밋이 작고 main에 깔끔하게 적용될 때
- **PR로 머지** — main 컨텍스트에 맞게 조정이 필요할 때
- **새로 작성** — main에서 코드 구조가 달라졌으면 같은 버그를 다른 방식으로 수정

자동화하지 않는 이유: hotfix는 임시 조치에 가깝고, main에서는 더 근본적인 수정이 필요할 수 있기 때문이다.

> 📍 tldraw 적용: `.study/01-dev-infra.md` — `trigger-production-build.yml`, `trigger-dotcom-hotfix.yml`, `trigger-sdk-hotfix.yml`

## GitHub Actions 권한 체계 — GITHUB_TOKEN vs PAT vs GitHub App

워크플로우에서 GitHub API를 호출할 때 사용하는 인증 수단은 세 가지가 있다.

| 방식 | 특징 | 한계 |
|------|------|------|
| **GITHUB_TOKEN** | Actions가 자동 제공, 해당 레포 범위 | 이 토큰으로 만든 이벤트(PR, push)는 **다른 워크플로우를 트리거하지 못함** (무한 루프 방지) |
| **PAT (Personal Access Token)** | 개인 계정에 묶인 토큰 | 특정 사람에 의존, 팀 환경에서 부적절 |
| **GitHub App 토큰** | 서비스 계정처럼 동작, 강한 권한 | App 등록 필요, 토큰이 최대 1시간 수명이라 동적 발급 필수 |

### GitHub App 토큰 발급 패턴

시크릿에 저장하는 건 토큰이 아니라 **App의 private key**. 워크플로우 실행 시 이 키로 임시 토큰을 발급받는다:

```yaml
- name: Generate a token
  id: generate_token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.HUPPY_APP_ID }}
    private-key: ${{ secrets.HUPPY_APP_PRIVATE_KEY }}

# 이후 단계에서 사용
- uses: actions/checkout@v6
  with:
    token: ${{ steps.generate_token.outputs.token }}
```

자동화가 고도화된 레포(PR 자동 생성, 자동 머지, 다른 워크플로우 트리거 등)에서는 이 패턴이 사실상 필수다.

> 📍 tldraw 적용: huppy-bot GitHub App — hotfix, 릴리스 등 자동화 전반에서 사용

## GitHub Required Checks와 `mergeable_state`

### Required checks

GitHub 브랜치 보호 규칙(Branch Protection Rules)에서 설정하며, **코드가 아니라 GitHub UI**(Settings → Branches → Branch protection rules)에서 관리한다.

- 워크플로우 YAML의 job `name`이 required check 이름과 일치하면 자동 매칭
- 해당 job이 정상 종료(exit code 0)하면 pass, 실패하면 fail
- 등록된 required checks가 **전부 pass**해야 PR을 머지할 수 있음

### `mergeable_state` — PR의 종합 머지 가능 상태

GitHub이 required checks, 충돌 여부 등을 종합하여 PR에 부여하는 상태값:

| 상태 | 의미 |
|------|------|
| `clean` | 모든 checks 통과, 충돌 없음 → 머지 가능 |
| `unstable` | 일부 checks 실패 |
| `dirty` | 머지 충돌 있음 |
| `blocked` | 보호 규칙에 의해 차단 (리뷰 미승인 등) |
| `unknown` | 아직 계산 중 |

워크플로우에서 다른 CI의 완료를 기다려야 할 때, 개별 check run을 추적하는 대신 이 값을 **폴링**하면 간단하다.

> 📍 tldraw 적용: `trigger-dotcom-hotfix.ts` — hotfix PR 생성 후 `mergeable_state`를 15분간 폴링하여 자동 머지

## React 컴포넌트 as SDK

"SDK"라고 하면 순수 로직 라이브러리를 떠올리지만, React 생태계에서는 **UI 컴포넌트 자체가 SDK**가 된다. npm 패키지로 배포된 React 컴포넌트를 import하고 렌더링하면 완전한 기능이 임베딩되는 패턴.

```tsx
// 소비자 코드 — 에디터 전체가 컴포넌트 하나로 임베딩됨
import { Tldraw } from 'tldraw'
import 'tldraw/tldraw.css'

function MyApp() {
  return <Tldraw />
}
```

이 패턴이 성립하는 이유:

- **React 컴포넌트 = 재사용 가능한 빌딩 블록**: 로직 + UI + 스타일이 하나의 단위로 캡슐화됨
- **npm 배포 가능**: `package.json`의 exports로 진입점을 정의하면 어떤 React 앱에서든 import 가능
- **CSS 분리 배포**: `tldraw.css`를 별도 import하게 하여 스타일도 패키지에 포함
- **Context + Hooks로 내부 상태 노출**: 소비자가 `useEditor()` 같은 hook으로 내부 API에 접근 가능

동일한 패턴을 쓰는 라이브러리들: Monaco Editor (VS Code 에디터), Slate/Lexical (리치 텍스트), Google Maps React SDK, React-PDF 등.

비유하면 **Google Maps SDK**와 같다. 지도 UI가 포함된 SDK인데 `<GoogleMap />` 컴포넌트 하나로 임베딩하고, 마커/오버레이를 커스텀할 수 있다. tldraw는 지도 대신 **무한 캔버스**를 제공하는 것.

> 📍 관련: `.study/03-architecture-patterns.md` — 레이어드 패키지 설계
> 📍 관련: `.study/05-api-design.md` — Progressive disclosure API 패턴

---

## TypeScript 함수 오버로드

같은 이름의 함수에 여러 호출 시그니처를 정의하는 TypeScript 문법. **타입 선언(시그니처)이 위에 여러 개** 오고, **구현부가 맨 마지막에 하나** 온다.

```ts
// 시그니처 1 — 타입 선언만, JS로 컴파일하면 사라짐
function useValue<Value>(value: Signal<Value>): Value

// 시그니처 2 — 타입 선언만
function useValue<Value>(name: string, fn: () => Value, deps: unknown[]): Value

// 구현부 — 런타임에 이것만 존재
function useValue() {
    const args = arguments
    if (args.length === 3) { /* 시그니처 2 */ }
    else { /* 시그니처 1 */ }
}
```

**규칙:**
- 시그니처들이 연속으로 와야 하고, 구현부가 반드시 맨 마지막
- 호출 시 TypeScript 컴파일러가 인자의 타입/개수를 보고 **위에서 아래로** 시그니처를 매칭
- 매칭은 컴파일 타임에만 일어남. 런타임에는 구현부 하나에서 `args.length` 등으로 직접 분기

Java의 메서드 오버로딩과 비슷하지만, 런타임 구현은 하나뿐이고 타입 시스템에서만 분기되는 것이 차이점.

> 📍 사용 예: packages/state-react/src/lib/useValue.ts — 시그널 직접 구독 vs computed 생성 두 가지 오버로드

---

## React Concurrent Mode

React 18부터 도입된 **렌더링을 쪼개서 실행하는 모드**.

```
기존 (동기 렌더링):
  렌더 시작 ────────────────────── 렌더 끝
  컴포넌트 1000개를 한 번에 쭉 렌더. 그 동안 유저 입력 멈춤.

Concurrent mode:
  렌더 시작 ── 중간에 끊음 ── 유저 입력 처리 ── 렌더 재개 ── 렌더 끝
```

렌더링 **계산**을 중간에 멈추고 더 급한 일(클릭, 타이핑)을 먼저 처리한 뒤 이어서 렌더한다. 단, **DOM 커밋은 한 번에** 한다 — "부모만 업데이트되고 자식은 안 된" 화면이 나오진 않는다.

- **렌더링 (계산)** — 쪼개서 실행 가능 (중간에 양보 가능)
- **커밋 (DOM 반영)** — 항상 한 번에 반영

렌더링 계산이 쪼개지기 때문에, 그 틈에 외부 값이 바뀌면 tearing 문제가 발생할 수 있다.

---

## Tearing — concurrent mode에서의 외부 상태 불일치

렌더링 계산이 쪼개져서 실행되는 틈에 외부 값이 바뀌어서, **같은 렌더 안에서 컴포넌트마다 다른 값으로 계산되는 현상**.

```
렌더 계산 시작 (store.color = "red")
  <Parent>  → "red"로 계산
  <ChildA>  → "red"로 계산
  --- time slice 양보 → 이 틈에 store.color = "blue" ---
  <ChildB>  → "blue"로 계산   ← 찢어짐!
렌더 끝 → DOM에 커밋
  → Parent는 red, ChildB는 blue로 화면에 반영됨
```

React가 관리하는 `useState`는 이 문제가 없다 — state가 바뀌면 React가 알고 있으니 진행 중인 렌더를 버리고 새 값으로 처음부터 다시 시작할 수 있다. React를 통해서만 값이 바뀌므로 렌더 도중에 값이 바뀌는 일 자체가 없다.

외부 상태(시그널, Redux store 등)는 React가 모르기 때문에 렌더를 버려야 하는지 판단을 못 한다. 그래서 `useSyncExternalStore`가 필요하다.

---

## useSyncExternalStore — React 외부 상태 연결 브릿지

React 18에서 추가된 hook. **React 바깥의 외부 저장소를 React state처럼 안전하게 연결**하는 공식 방법.

```ts
useSyncExternalStore(subscribe, getSnapshot)
```

| 파라미터 | 역할 |
|---------|------|
| `subscribe(callback)` | "값이 바뀌면 이 callback 호출해줘" — 구독 등록 |
| `getSnapshot()` | "현재 값이 뭐야?" — React가 필요할 때 호출 |

React가 렌더 도중에 `getSnapshot()` 반환값이 달라진 것을 감지하면, 해당 렌더를 버리고 새 값으로 동기적으로 다시 렌더한다. 이를 통해 tearing을 방지한다.

`useState` + `useEffect` 조합으로도 비슷하게 할 수 있지만, concurrent mode에서의 tearing을 방지할 수 없다. tldraw 시그널, Redux, Zustand 등 외부 상태 라이브러리들이 모두 내부적으로 이 hook을 사용한다.

> 📍 관련: `.study/06-state-management.md` — useValue에서의 사용
> 📍 관련: `.study/02-ui-architecture.md` — 반응형 UI

---

## Proxy apply trap — 함수 호출 가로채기

JavaScript `Proxy`의 `apply` 트랩은 함수가 호출될 때 가로챈다. 함수의 프로퍼티 접근이 아니라 **함수 호출(`()`) 자체**를 인터셉트한다.

```ts
function original(x) { return x * 2 }

const proxied = new Proxy(original, {
  apply(target, thisArg, args) {
    console.log('호출 전')
    const result = target.apply(thisArg, args)
    console.log('호출 후')
    return result
  }
})

proxied(5)  // "호출 전" → 10 → "호출 후"
```

**핵심**: `new Proxy(fn, { apply })` 후에도 `proxied`는 여전히 함수처럼 보인다. `typeof proxied === 'function'`이고, 프로퍼티 접근(`proxied.name`, `proxied.displayName`)은 원본 함수로 그대로 전달된다. 함수 호출만 가로챌 뿐이다.

tldraw의 `track`이 이걸 이용한다. React 함수형 컴포넌트는 그냥 함수이므로, React가 `Component(props)`를 호출하면 `apply` 트랩이 가로채서 `useStateTracking`으로 감싼다. React를 수정하거나 해킹한 게 아니라, JS 언어 레벨의 함수 인터셉트다.

> 📍 관련: `.study/06-state-management.md` — track HOC 구현 원리 상세

---

## TypeScript 데코레이터 — 클래스 멤버를 감싸는 선언적 문법

클래스의 메서드/프로퍼티/클래스 자체를 **감싸서 동작을 바꾸는 문법**. `@` 기호를 붙인다. 본질은 **함수 호출의 축약**이다.

```ts
// 데코레이터는 그냥 함수다
function log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value
  descriptor.value = function (...args: any[]) {
    console.log(`${key} 호출됨`)
    return original.apply(this, args)
  }
}

class Calculator {
  @log
  add(a: number, b: number) { return a + b }
}
// calc.add(1, 2) → "add 호출됨" 출력 후 3 반환
```

데코레이터 없이 동일한 코드:

```ts
class Calculator {
  add(a: number, b: number) { return a + b }
}
log(Calculator.prototype, 'add',
  Object.getOwnPropertyDescriptor(Calculator.prototype, 'add'))
```

### 데코레이터 종류

```ts
@classDecorator           // 클래스 자체
class Foo {
  @propertyDecorator      // 프로퍼티
  name: string

  @methodDecorator        // 메서드
  doSomething() {}
}
```

### 데코레이터 vs Proxy

둘 다 "원래 동작을 가로채서 바꾼다"는 점에서 비슷하지만 타이밍이 다르다.

| | 데코레이터 | Proxy |
|--|-----------|-------|
| **언제** | 클래스 정의 시점 (1번) | 런타임에 아무 때나 |
| **대상** | 클래스/메서드/프로퍼티 | 아무 객체 |
| **방식** | 메서드를 미리 교체해둠 | 접근할 때마다 trap 실행 |

tldraw에서 둘 다 사용: `@computed`는 데코레이터(클래스 정의 시 메서드를 Computed 시그널로 교체), `track()`은 Proxy(컴포넌트 함수 호출을 런타임에 가로채기).

> 📍 관련: `.study/06-state-management.md` — @computed 데코레이터, track HOC

## 리플로우(Reflow) — 브라우저 레이아웃 재계산

DOM 요소의 크기/위치가 바뀌면 브라우저가 영향받는 모든 요소의 레이아웃을 다시 계산하는 과정. "Layout" 또는 "Reflow"라 부른다.

한 요소의 높이가 바뀌면 → 부모 높이도 바뀔 수 있음 → 형제도 밀려남 → 그 부모도... 이렇게 **DOM 트리를 타고 연쇄적으로 전파**되어 비용이 큼.

CSS `contain` 속성으로 이 전파 체인을 끊을 수 있다:
- `contain: size` — 내부 콘텐츠가 바뀌어도 내 크기는 안 변함
- `contain: layout` — 내부 레이아웃이 외부에 영향 안 줌
- `contain: strict` — size + layout + style + paint 전부 (완전 격리)

브라우저는 이런 최적화를 자동으로 못 한다. "자식이 부모 크기에 영향을 주는지"는 코드 의도를 알아야 판단 가능하므로, 개발자가 명시적으로 선언해야 한다.

> 📍 관련: `.study/10-canvas-rendering.md` — CSS Containment로 리플로우 격리, `.study/07-performance.md` — CSS Containment

## GPU 합성 레이어(Compositing Layer)

브라우저가 특정 DOM 요소를 별도의 GPU 텍스처로 승격시켜, transform/opacity 변경 시 CPU의 레이아웃/페인트를 건너뛰고 GPU 합성만으로 처리하는 메커니즘.

**확정 승격 조건**: `translate3d()`, `will-change: transform`, `<video>`, `<canvas>`, `filter` 등
**힌트(브라우저 판단)**: `contain: strict`, `content-visibility: auto`, 2D transform 빈번 변경

GPU 레이어는 공짜가 아니다. 레이어당 VRAM 할당(요소 크기 × 4bytes) + 매 프레임 합성 비용. 요소 1000개에 `will-change: transform`을 걸면 GPU 메모리가 폭발할 수 있다. 필요한 곳에만 선별적으로 적용해야 함.

> 📍 관련: `.study/10-canvas-rendering.md` — GPU 가속 전략, `.study/07-performance.md` — GPU 가속

## CSS `transform: matrix(a,b,c,d,e,f)` — 2D 아핀 변환 행렬

이동(translate), 회전(rotate), 스케일(scale)을 6개 숫자의 행렬 하나로 표현하는 CSS 속성.

```
matrix(a, b, c, d, e, f)
a = scaleX × cos(θ)    c = -scaleY × sin(θ)    e = translateX
b = scaleX × sin(θ)    d = scaleY × cos(θ)     f = translateY
```

`translate()`, `rotate()`, `scale()`을 따로 쓰는 것과 달리, matrix끼리는 행렬 곱셈으로 합성 가능. 부모-자식 변환 체인을 곱셈 한 번으로 풀 수 있어서 좌표계 변환이 필요한 캔버스/게임 엔진에서 많이 쓴다.

> 📍 관련: `.study/10-canvas-rendering.md` — CSS matrix() 도형 배치

## R-tree — 2D 공간 검색 자료구조

"이 영역 안에 뭐가 있지?"를 O(log n)에 찾기 위한 트리. B-tree의 2D 버전.

가까운 객체끼리 최소 바운딩 사각형(MBR)으로 묶고, 그 사각형들을 다시 묶어 계층 구조를 만든다. 쿼리 시 영역과 안 겹치는 그룹은 통째로 스킵하여 개별 비교를 줄인다.

- **검색**: 루트부터 겹치는 노드만 타고 내려감 → O(log n)
- **벌크 로딩**: STR 알고리즘 — X축 정렬 → 슬라이스 → Y축 정렬 → 타일 분할
- **개별 삽입**: "면적 증가가 가장 적은 노드"를 선택하며 내려감. 노드가 꽉 차면 분할(split)

JS 구현체로 **RBush** 라이브러리가 있으며 tldraw가 사용 중. 지도, 게임, 캔버스 에디터에서 공간 쿼리에 널리 쓰인다.

> 📍 관련: `.study/10-canvas-rendering.md` — R-tree 기반 culling, `.study/07-performance.md` — R-tree 공간 인덱싱
