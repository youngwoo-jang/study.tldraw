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

### 버전 업데이트

같은 버전을 두 번 올릴 수 없다. 반드시 버전을 올려야 함.

```bash
npm version patch   # 4.5.3 → 4.5.4 (버그 픽스)
npm version minor   # 4.5.3 → 4.6.0 (기능 추가)
npm version major   # 4.5.3 → 5.0.0 (호환성 깨지는 변경)
npm publish
```

`npm version`은 package.json version 수정 + git commit + git tag 생성.

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
