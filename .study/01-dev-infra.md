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

## 퍼블리시 파이프라인 상세 — 10개 워크플로우의 역할과 관계

퍼블리시 관련 워크플로우는 10개이며, 3개 그룹으로 나뉜다.

> 📍 .github/workflows/publish-*.yml
> 📍 .github/workflows/bump-versions.yml, get-changelog.yml, update-release-notes.yml
> 📍 internal/scripts/publish-*.ts, extract-draft-changelog.tsx

### 그룹 1: 정식 릴리스

| 워크플로우 | 트리거 | npm dist tag | 특징 |
|---|---|---|---|
| `publish-new` | 수동, `production` 브랜치 | `latest` | minor/major/override 선택, 사전 draft release 필수 |
| `publish-patch` | `v*.*.x` 브랜치 push (자동) | `latest` / `revision` | 패치 버전 자동 bump, 체인지로그 자동 생성 |
| `publish-manual` | 수동, 버전 번호 입력 | 기존 태그 유지 | 버전 bump/체인지로그 없이 재퍼블리시만 |

- `publish-new`와 `publish-patch`는 완료 후 `publish-templates`를 체인으로 호출
- `publish-manual`은 npm 퍼블리시가 실패했을 때 복구용. 버전 bump이나 체인지로그 생성 없이 해당 버전을 다시 올리기만 함

### 그룹 2: 프리릴리스

| 워크플로우 | 트리거 | npm dist tag | 특징 |
|---|---|---|---|
| `publish-canary` | `main` push (자동) | `canary` | 최신 개발 버전 |
| `publish-canary` | `production` push (자동) | `next` | 정식 릴리스 직전 버전 + tldraw-desktop에 webhook 알림 |
| `publish-branch` | PR에 `publish-packages` 라벨 | `internal` | PR 코드를 npm에서 테스트 가능, 완료 후 PR에 버전 코멘트 |

- 3개 모두 같은 스크립트(`publish-prerelease.ts`)를 인자만 바꿔서 사용
- `publish-branch`는 라벨을 자동 제거하고, PR 코멘트로 설치 가능한 버전 번호를 알려줌

### 그룹 3: 보조 워크플로우

| 워크플로우 | 트리거 | 역할 |
|---|---|---|
| `publish-templates` | 다른 워크플로우에서 `workflow_call` | `templates/` 디렉토리를 별도 GitHub 리포로 내보냄 (`create-tldraw`용) |
| `publish-editor-extensions` | main/production push (자동) | VSCode 확장을 VSCE + OVSX에 배포, main=staging / production=production |
| `bump-versions` | 수동, main 브랜치 | npm 최신 버전으로 main의 모든 package.json 동기화 |
| `get-changelog` | 수동 | 체인지로그 초안을 GitHub Artifacts로 업로드 |
| `update-release-notes` | 수동 | Claude Code가 docs 릴리스 노트 자동 작성 → PR 생성 |

### Concurrency 제어

```yaml
concurrency:
  group: npm-publish
```

`publish-new`, `publish-patch`, `publish-manual` 3개가 같은 `npm-publish` concurrency 그룹을 공유하여 동시 실행을 방지한다. `publish-canary`는 별도 태그(`canary`/`next`)로 배포되므로 이 그룹에 포함되지 않는다.

`publish-editor-extensions`는 자체 `vscode-extension-publish` concurrency 그룹을 사용.

### 전체 흐름

```
개발자가 main에 머지
  ├→ publish-canary (canary 태그 프리릴리스)
  └→ publish-editor-extensions (VSCode staging 배포)

main → production 머지
  ├→ publish-canary (next 태그 프리릴리스 + 데스크톱 앱 알림)
  └→ publish-editor-extensions (VSCode production 배포)

수동 publish-new (production 브랜치)
  └→ 버전 bump → npm 배포 → GitHub Release 공개 → publish-templates
     └→ 자동으로 bump-versions 트리거 (main 동기화)

핫픽스: v*.*.x 브랜치에 push
  └→ publish-patch 자동 실행

릴리스 후
  └→ update-release-notes로 docs 릴리스 노트 PR 자동 생성
```

### YAML은 얇게, 로직은 TypeScript로

워크플로우 YAML은 트리거/환경/시크릿/concurrency만 담당하고, 실제 비즈니스 로직은 전부 `internal/scripts/`의 TypeScript 스크립트로 구현한다:

```yaml
# 워크플로우가 하는 건 이것뿐
- name: Publish
  run: yarn tsx ./internal/scripts/publish-new.ts --bump ${{ inputs.bump_type }}
```

```
internal/scripts/
├── publish-new.ts              ← minor/major 릴리스
├── publish-patch.ts            ← 패치 릴리스
├── publish-prerelease.ts       ← canary/next/internal
├── publish-manual.ts           ← 재발행
├── bump-versions.ts            ← 버전 동기화
├── export-template.ts          ← 템플릿 내보내기
├── publish-editor-extensions.ts ← VSCode 확장
├── extract-draft-changelog.tsx  ← 체인지로그 생성
└── lib/publishing.ts           ← 공통 함수
```

이렇게 하는 이유:
- YAML로 복잡한 로직(조건분기, 에러 핸들링, API 호출)을 짜면 디버깅이 어려움
- **로컬에서 직접 실행 가능** — CI 중간에 실패하면 로컬에서 스크립트를 돌려 복구 (publish-new.ts 주석: `[IF THIS STEP FAILS, RUN THE publish-manual.ts script locally]`)
- `lib/publishing.ts`의 공통 함수를 여러 스크립트가 재사용

## 체인지로그 & GitHub Release & 릴리스 노트 — 세 가지 릴리스 문서

tldraw는 릴리스 시 세 종류의 문서를 생성한다.

### 1) GitHub Release (github.com/tldraw/tldraw/releases)

GitHub 저장소의 Releases 페이지에 올라가는 것. 생성 방식이 minor/major vs patch에서 다르다.

**minor/major (`publish-new`)** — 사전에 **수동으로 Draft Release를 만들어둬야 함**

```typescript
// publish-new.ts — draft release가 없으면 에러
const draftRelease = await getDraftRelease(nextVersion, octokit)
// ... npm 배포 완료 후 draft → published로 전환
await octokit.rest.repos.updateRelease({
    release_id: draftRelease.id,
    draft: false,
    tag_name: gitTag,
})
```

릴리스 노트 본문은 사람(또는 update-release-notes의 Claude)이 미리 작성한다.

**patch (`publish-patch`)** — 체인지로그를 **자동 생성**하여 GitHub Release를 바로 생성

```typescript
// publish-patch.ts
const changelog = await extractChangelog(prevTag, 'HEAD')
await octokit.repos.createRelease({
    tag_name: tag,
    body: changelog,  // 자동 생성
    draft: false,
})
```

### 2) 자동 체인지로그 (`extract-draft-changelog.tsx`)

git 커밋을 순회하면서 마크다운을 생성하는 스크립트. 다음 규칙으로 필터링/분류한다:

- `packages/` 폴더를 건드린 커밋만 포함 (내부 패키지 `dotcom-shared`, `worker-shared` 제외)
- PR 본문에서 `### Release Notes`와 `### API Changes` 섹션만 추출
- PR 본문의 체크박스(`- [x] \`bugfix\``)로 change type 분류
- 카테고리별 그룹핑: Bug Fixes / Improvements / Features / API Changes / Other

> 📍 internal/scripts/extract-draft-changelog.tsx

### 3) docs 릴리스 노트 (tldraw.dev/releases)

`apps/docs/content/releases/v4.5.0.mdx` 같은 형태의 MDX 파일. SDK 사용자를 위해 **코드 예시와 마이그레이션 가이드**를 포함하는 문서.

```
구조:
- 프론트매터 (title, description, date, keywords)
- 서두 요약 (한 문단)
- What's new — 주요 신기능 상세 + 코드 예시
- API changes — 추가/변경/삭제된 공개 API
- Improvements — 성능, UX 개선
- Bug fixes — 수정된 버그 목록
```

`next.mdx`에 다음 릴리스용 내용을 계속 누적하다가, 릴리스 시 `v4.6.0.mdx`로 아카이브.

`update-release-notes` 워크플로우에서 **Claude Code가 CI에서 실행**되어 자동 작성:

```yaml
- name: Run update-release-notes skill
  run: claude --skip-permissions --print "/update-release-notes"
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 세 가지의 비교

| | GitHub Release | 자동 체인지로그 | docs 릴리스 노트 |
|---|---|---|---|
| **위치** | github.com releases | GitHub Release body | tldraw.dev/releases |
| **형태** | 마크다운 | 커밋 기반 목록 | MDX (코드 예시 포함) |
| **생성** | minor: 수동 draft, patch: 자동 | `extract-draft-changelog.tsx` | Claude Code + 수동 리뷰 |
| **대상** | 개발자 전반 | 개발자 (변경사항 전체) | SDK 사용자 (주요 변경만) |

> 📍 apps/docs/content/releases/*.mdx
> 📍 internal/scripts/extract-draft-changelog.tsx
> 📍 .github/workflows/update-release-notes.yml

## CI 파이프라인 — `checks.yml`

핵심 CI 워크플로우. PR, merge group, main push에서 트리거. 2개 병렬 Job으로 구성.

> 📍 .github/workflows/checks.yml

### test Job — 코드 품질 검증

ARM 16코어 러너(`ubuntu-latest-16-cores-arm-open`)에서 순차 실행:

```
constraints → dedupe → check-packages → circular-deps
→ [캐시 복원] → build-types → bundle-size → check-pr-template
→ api-check → lint → build-i18n + git diff → test-ci
```

- `yarn constraints` — 워크스페이스 간 의존성 버전 통일 검사
- `yarn dedupe --check` — yarn.lock 중복 의존성 감지
- `yarn check-circular-deps` — 패키지 간 순환 참조 검사
- `yarn build-types` — TypeScript 타입 체크
- `yarn lazy check-bundle-size` — SDK 번들 크기 검증
- `yarn api-check` — 공개 API 선언 일관성 검사
- `yarn lint` — OxLint 코드 린트
- `yarn build-i18n` + `git diff` — 번역 파일 빌드 후 커밋된 상태와 일치하는지 확인
- `yarn test-ci` — 전체 유닛 테스트

### build Job — 빌드 검증

```
[캐시 복원] → yarn build-package
```

빌드가 되는지만 확인 (배포 목적 아님). test와 병렬 실행.

### 빌드 캐시 설정

`.lazy`, `.tsbuild`, `.tsbuildinfo`를 GitHub Actions 캐시에 저장하여 증분 빌드 지원.

```yaml
key: test-tools-${{ runner.os }}-${{ hashFiles('yarn.lock', ...) }}-${{ github.sha }}
restore-keys: |
  test-tools-${{ runner.os }}-${{ hashFiles('yarn.lock', ...) }}-
```

정확한 SHA 매칭 실패 시 같은 의존성의 가장 최근 캐시를 fallback으로 복원.

> 📍 관련 개념: `.study/00-dev-concepts.md` — CI 러너와 캐싱 전략

## 커스텀 액션 — `.github/actions/setup`

모든 워크플로우에서 공통으로 쓰는 Node + yarn 설정을 재사용 가능한 조각으로 추출.

> 📍 .github/actions/setup/action.yml

```yaml
steps:
  - run: npm i -g corepack           # corepack 활성화
  - uses: actions/setup-node@v6       # Node 24.13.1 설치
    with: { node-version: 24.13.1 }
  - uses: actions/cache/restore@v5    # yarn 캐시 복원 (읽기만)
  - run: yarn install --immutable     # 의존성 설치
  - uses: actions/cache/save@v5       # main push일 때만 캐시 저장
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

핵심: **캐시 저장을 main에서만** 수행. PR 브랜치마다 ~1GB yarn 캐시를 저장하면 레포당 10GB 한도를 금방 초과하므로, PR은 main의 캐시를 읽어서 쓰기만 한다. `actions/cache`를 `restore`/`save`로 분리하는 패턴은 대형 모노레포에서 흔한 최적화.

- `yarn install --immutable` — yarn.lock과 다른 의존성 설치 시 에러 (CI에서 lock 파일 변경 방지)
- `YARN_ENABLE_HARDENED_MODE: 0` — 보안 검사 비활성화, ~15초 절약

함수 추출과 같은 개념. 중복 step들을 커스텀 액션으로 빼서 test/build Job에서 `uses: ./.github/actions/setup`으로 호출.

## Claude 봇 — `claude.yml`

PR/이슈에서 `@claude`를 멘션하면 Claude가 자동 응답하는 워크플로우.

> 📍 .github/workflows/claude.yml

```yaml
on:
  issue_comment: [created]
  pull_request_review_comment: [created]
  pull_request_review: [submitted]
  issues: [opened, assigned]
```

`anthropics/claude-code-action@v1` 공식 GitHub Action을 사용. 레포를 checkout → 멘션된 코멘트 읽기 → Anthropic API로 Claude 호출 → PR/이슈에 코멘트로 응답.

- `ANTHROPIC_API_KEY`가 필수 (GitHub Secrets에 등록)
- Max 플랜(구독)으로는 사용 불가 — API 키(종량제)만 지원
- 한때(2025.7) OAuth 토큰 지원이 발표되었으나, 2026.1에 자동화 용도가 차단됨
- 퍼블릭 레포에서는 아무나 `@claude` 멘션 가능하므로 API 비용 주의

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

## Playwright E2E 워크플로우 (4개)

Playwright E2E 테스트를 4개 워크플로우로 분리하여, 목적별로 러너 크기와 설정을 다르게 운용한다.

> 📍 .github/workflows/playwright-dotcom.yml
> 📍 .github/workflows/playwright-examples.yml
> 📍 .github/workflows/playwright-perf.yml
> 📍 .github/workflows/playwright-update-snapshots.yml

### 공통 구조

```
check (paths-filter) → setup (Node + Yarn) → Playwright 캐시 → 빌드 → 테스트 → 리포트
```

- 트리거: `pull_request`, `merge_group`, `push(main)` (update-snapshots만 `labeled`)
- `dorny/paths-filter`로 관련 파일 변경 시에만 실행 (각 워크플로우가 자기한테 관련된 경로를 직접 정의)
- Playwright 브라우저 캐시: `~/.cache/ms-playwright`를 `OS-arch-playwright-{version}` 키로 캐싱
- 리포트 이중화: GitHub Artifact(30일) + S3(`playwright.tldraw.xyz`), 성공/실패 무관(`if: always()`)

> 📍 관련 개념: `.study/00-dev-concepts.md` — `dorny/paths-filter`, CI 러너와 캐싱 전략

### 워크플로우별 차이

| | dotcom | examples | perf | update-snapshots |
|---|---|---|---|---|
| **러너 코어** | 8 | 32 | 16 | 8 |
| **타임아웃** | 20분 | 20분 | 20분 | 60분 |
| **인증** | Clerk | 없음 | 없음 | 없음 |
| **Fork PR** | 차단 | 허용 | 허용 | N/A |
| **병렬화** | 순차(파일 내) | 완전병렬 | 완전병렬 | N/A |
| **리트라이** | 2 | 1 | 1 | N/A |
| **모바일 테스트** | X | O (Pixel 5) | O | O |
| **커밋 생성** | X | X | X | O (자동) |

### playwright-dotcom — 프로덕트 검증

- `e2e-dotcom` GitHub Environment로 시크릿 스코핑
- Clerk 인증: 빌드 시 `VITE_CLERK_PUBLISHABLE_KEY`, 테스트 시 `.dev.vars`에 Clerk 키 파일 생성
- `e2e-x10` 라벨 분기: 라벨 있으면 `--repeat-each=10`으로 각 테스트 10회 반복 (flaky 테스트 탐지용)
- `fullyParallel: false` — 클립보드 같은 공유 리소스 보호를 위해 파일 내 순차 실행
- Fork PR 차단 (`fork == false`) — Clerk 시크릿 접근 불가하므로

### playwright-examples — SDK 예제 검증

- 32코어 (가장 큰 러너) — 다양한 예제 시나리오로 테스트 볼륨이 가장 큼
- `fullyParallel: true` — 최대 병렬화
- 프로젝트: `chromium` + `Mobile Chrome` (Pixel 5)
- 스크린샷 비교: `maxDiffPixelRatio: 0.0001`, `threshold: 0.01` (매우 엄격)
- 웹서버 2개: 예제 앱(5420) + bemo-worker(8989, 멀티플레이어 백엔드)

### playwright-perf — 성능 측정

- examples와 거의 동일하되 `e2e/perf/` 디렉토리의 성능 전용 테스트만 실행
- `PERFORMANCE_ANALYTICS_ENABLED: true` 환경변수 추가
- 16코어 — 성능 측정이므로 32코어만큼 불필요, 일관된 환경이 중요

### playwright-update-snapshots — 스냅샷 자동 갱신

테스트가 아닌 **자동화 도구**. `update-snapshots` 라벨을 PR에 붙이면 트리거.

1. 라벨 즉시 제거 (`actions-ecosystem/action-remove-labels`)
2. PR head ref로 체크아웃
3. Playwright 브라우저 매번 설치 (캐시 없이, 일관성 보장)
4. `yarn e2e --update-snapshots` (기준 스냅샷 재생성)
5. `huppy-bot[bot]`으로 자동 커밋+푸시

```bash
git config user.name 'huppy-bot[bot]'
git commit --no-verify -m '[automated] update snapshots'
git pull --rebase && git push
```

- 권한: `contents: write`, `pull-requests: write`
- `git add -A && git reset --hard HEAD` — husky 훅이 예상치 못한 변경을 만드는 경우 방어

> 📍 상세 분석: `.study/08-testing.md`

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

## `deploy-dotcom.yml` — 메인 앱 배포 워크플로우

하나의 워크플로우가 브랜치에 따라 preview/staging/production 3가지 환경을 결정하고, 각 환경에 격리된 풀스택(프론트 + Workers + DB)을 통째로 배포한다.

> 📍 .github/workflows/deploy-dotcom.yml
> 📍 internal/scripts/deploy-dotcom.ts

### 환경 결정

```yaml
TLDRAW_ENV: ${{ (github.ref == 'refs/heads/production' && 'production') ||
                (github.ref == 'refs/heads/main' && 'staging') || 'preview' }}
```

- `production` 브랜치 push → production 환경
- `main` 브랜치 push → staging 환경
- PR + `dotcom-preview-please` 라벨 → preview 환경 (opt-in)

GitHub environment(`deploy-production` vs `deploy-staging`)별로 시크릿이 분리되어 있다.

### 배포되는 서비스 (7개)

| 서비스 | 플랫폼 | 역할 |
|--------|--------|------|
| dotcom client (SPA) | Vercel | 프론트엔드 React 앱 |
| sync-worker (multiplayer) | Cloudflare Workers | 실시간 협업 백엔드 + DB 마이그레이션 |
| asset-upload-worker | Cloudflare Workers | 파일 업로드 |
| image-resize-worker | Cloudflare Workers | 이미지 최적화 |
| tldrawusercontent-worker | Cloudflare Workers | 사용자 콘텐츠 |
| health-worker | Cloudflare Workers | 헬스체크 + Discord 알림 |
| zero-cache | Fly.io | 로컬-퍼스트 동기화 캐시 |

### Workers 의존성 그래프와 배포 순서

```
multiplayer ─┬─→ tldrawusercontent (service binding 의존)
             └─→ image-resize (service binding 의존)
asset-upload ──→ (독립)
health ────────→ (독립)
```

`deployWorkersInDependencyOrder()`가 토폴로지 정렬로 배포 순서를 관리한다.

### 배포 파이프라인 (2-phase 패턴)

```
1. prebuild         → 패키지 에셋 준비 (CSS, 아이콘, 폰트)
2. dotcom SPA 빌드  → Sentry 릴리스 + Vite 빌드 + 소스맵 업로드 + R2 에셋 병합
3. Workers dry run   → 5개 전부 병렬로 빌드만 (배포 가능 여부 확인)
   ──── point of no return ────
4. Workers 실배포    → 의존성 순서대로 순차 배포
5. Vercel SPA 배포   → Workers 다 뜬 후 배포 (프론트가 먼저 뜨면 없는 Worker 호출)
6. alias + GitHub Deployment → 프리뷰 URL 매핑 + PR에 배포 링크 표시
```

dry run으로 전부 통과 확인 후 실배포하는 all-or-nothing 전략.

### PR 프리뷰 — 격리된 풀스택

프리뷰 시 모든 서비스에 `previewId` 접두사가 붙어 완전히 격리된다:

```typescript
env.ASSET_UPLOAD = `https://${previewId}-tldraw-assets.tldraw.workers.dev`
env.MULTIPLAYER_SERVER = `https://${previewId}-tldraw-multiplayer.tldraw.workers.dev`
env.IMAGE_WORKER = `https://${previewId}-images.tldraw.xyz`
```

- **Supabase 브랜치 DB**: PR마다 `pr-{number}` 격리 DB 자동 생성. `reset-preview-db` 라벨로 초기화 가능
- **Vercel alias**: `vercel alias set`으로 `{previewId}-preview-deploy.tldraw.com` 매핑
- **GitHub Deployment**: PR 페이지에 "View deployment" 링크 자동 표시

Vercel 기본 프리뷰를 안 쓰는 이유: 기본 프리뷰는 프론트엔드만 배포하므로 백엔드 변경사항을 테스트할 수 없다. Vercel은 CLI 도구(`vercel deploy --prebuilt`)로만 사용하고, 배포 순서 제어는 자체 스크립트가 담당.

> 📍 관련 개념: `.study/00-dev-concepts.md` — PR 프리뷰 배포, dry run, 에셋 병합
> 📍 에셋 병합 상세: `.study/09-resource-management.md` — R2 에셋 병합
> 📍 Zero 백엔드 상세: `.study/11-multiplayer-sync.md` — Zero 백엔드

## deploy-bemo.yml — 멀티플레이어 데모 서버 배포

bemo(tldraw 내부 명칭)는 SDK 데모/예제용 경량 멀티플레이어 서버. 프로덕션 sync-worker의 축소판.

> 📍 .github/workflows/deploy-bemo.yml
> 📍 apps/bemo-worker/

### 트리거 & 환경 결정

```
pull_request (모든 PR) → preview
push to main           → staging  (canary-demo.tldraw.xyz)
push to bemo-production → production (demo.tldraw.xyz)
```

`TLDRAW_ENV` 환경변수로 삼항 결정. `concurrency` 키로 같은 환경 동시 배포 방지.

### 배포 단계

1. checkout (서브모듈 포함) → setup (Node, yarn) → `yarn build-types` → `yarn tsx internal/scripts/deploy-bemo.ts`

실제 배포 로직은 워크플로우가 아니라 `internal/scripts/deploy-bemo.ts`에 있다 (wrangler dry-run → 실배포 → 도메인 매핑 → Sentry + Discord 알림).

### bemo 기술스택

| 계층 | 기술 | 역할 |
|------|------|------|
| 런타임 | Cloudflare Workers | 엣지 라우팅 (게이트웨이) |
| 상태 관리 | Durable Objects (`BemoDO`) | 방별 WebSocket + 동기화 상태 유지 |
| 저장소 | R2 | 문서 스냅샷 영속 저장 |
| 분석 | Analytics Engine | 사용량 데이터 수집 |
| 라우팅 | itty-router | 경량 HTTP 라우터 |
| 동기화 | @tldraw/sync-core | 자체 동기화 프로토콜 (프로덕션과 동일) |
| 빌드 | esbuild + wrangler | 번들링 + 배포 (번들 크기 제한 350KB) |
| 모니터링 | Sentry | 에러 트래킹 |

### bemo vs 프로덕션 sync-worker 비교

| | bemo | dotcom/sync-worker |
|---|---|---|
| 용도 | SDK 데모, 예제 앱 | tldraw.com 실서비스 |
| Durable Object | 1개 (`BemoDO`) | 5개 (File, User, Stats, Logger, PostgresReplicator) |
| 저장소 | 인메모리 + R2 | SQLite + R2 + PostgreSQL |
| 인증 | 없음 | 있음 |

> 📍 동기화 프로토콜 상세: `.study/11-multiplayer-sync.md` — tlsync 동기화 프로토콜
> 📍 관련 개념: `.study/00-dev-concepts.md` — 서버리스 vs Durable Objects

## `deploy-analytics.yml` — 쿠키 동의 확인 서비스 배포

이름은 "analytics"이지만 실제로는 **쿠키 동의(consent) 확인 서비스**를 배포하는 워크플로우다. Cloudflare Worker로 배포된다.

> 📍 .github/workflows/deploy-analytics.yml
> 📍 apps/analytics-worker/src/worker.ts

### 동작 원리

사용자의 IP 기반 국가 코드(`CF-IPCountry` 헤더)를 보고 쿠키 동의가 필요한 국가인지 판단하여 JSON으로 응답한다.

```json
{ "requires_consent": true, "country_code": "DE" }
```

동의 필요 국가: EU/EEA(GDPR), 영국(UK PECR), 스위스(FADP), 브라질(LGPD). 국가를 알 수 없으면 기본적으로 동의 필요로 처리한다.

### 별도 Worker로 분리한 이유

- tldraw.com은 Vercel에 배포되는 SPA → `CF-IPCountry` 헤더를 쓸 수 없음
- Cloudflare Worker에서는 CF가 자동으로 IP 기반 국가 헤더를 붙여줌
- `consent.tldraw.xyz`라는 별도 엔드포인트로 두면 `.tldraw.com`, `.tldraw.dev` 등 여러 사이트에서 공유 가능

### analytics 시스템 전체 구조

```
브라우저 → consent.tldraw.xyz (CF Worker)
        ← { requires_consent: true/false }
        → 동의 필요하면 배너 표시, 동의 시 analytics 활성화
```

연동 서비스: GA4, GTM, PostHog, HubSpot, Reo — 동의 여부에 따라 전부 on/off 제어된다.

> 📍 apps/analytics/src/index.ts — Analytics 클래스, 5개 서비스 통합 관리
> 📍 apps/analytics/src/utils/consent-check.ts — `shouldRequireConsent()`, consent.tldraw.xyz 호출

### 배포 트리거

- **PR**: path 필터 적용 (`apps/analytics-worker/**` 등 관련 파일 변경 시에만)
- **push**: `main` → staging, `production` → production으로 환경 분기
- `TLDRAW_ENV` 환경변수로 3단계(preview/staging/production) 분기

## `trigger-production-build.yml` — 프로덕션 배포 트리거 (릴리스 트레인)

production 브랜치에 `git commit-tree`로 머지 커밋을 만들어 배포를 트리거하는 워크플로우다.

> 📍 .github/workflows/trigger-production-build.yml

### 트리거 조건

| 트리거 | 상황 |
|--------|------|
| `hotfixes` 브랜치에 push | 긴급 핫픽스 배포 |
| workflow_dispatch (수동) | 원하는 ref를 지정해서 프로덕션 배포 (기본값: `main`) |

### 핵심 메커니즘: `git commit-tree`

일반적인 `git merge`가 아니라 저수준 Git 명령으로 직접 머지 커밋을 조립한다.

```bash
# 배포 대상의 트리(파일 스냅샷)를 그대로 사용하여 머지 커밋 생성
tree_hash=$(git show --quiet --pretty=format:%T "$TARGET_HASH")
commit=$(git commit-tree -m "$message" -p "$current_prod_hash" -p "$TARGET_HASH" "$tree_hash")
git update-ref refs/heads/production "$commit"
git push origin production
```

Git 히스토리상으로는 머지 커밋이지만, 실제 파일 내용은 배포 대상(main 또는 hotfix)의 것을 100% 그대로 사용한다. 병합이 아니라 교체이므로 **머지 충돌이 원천적으로 불가능**하다.

```
main:       A → B -----→ C ------→ D
                 ↘         ↘         ↘
production: X → M1 ----→ M2 -----→ M3
           (내용:B)    (내용:C)   (내용:D)
```

### 전체 흐름

```
1) hotfixes 브랜치에 push  또는  수동으로 target ref 지정
2) production 브랜치를 checkout
3) commit-tree로 머지 커밋 생성 (배포 대상의 트리로 교체)
4) git push origin production → deploy-dotcom.yml 트리거 → 실제 배포
5) git push origin production:hotfixes --force → hotfixes 브랜치 초기화
```

### 안전장치

- **중복 배포 방지**: hotfix 커밋이 이미 production에 포함되어 있으면 skip
- **concurrency**: `trigger-production` 키로 동시 실행 방지
- **GitHub App 토큰**: `huppy` 봇 앱 토큰 사용 — 일반 `GITHUB_TOKEN`으로 push하면 다른 워크플로우가 트리거되지 않는 제한을 우회

### Hotfix 경로

```
production에서 hotfixes 브랜치 분기 → 긴급 수정 push → 이 워크플로우가 production에 반영 → 배포
→ hotfixes 브랜치를 production으로 force push (초기화)
→ main에는 개발자가 수동으로 cherry-pick 또는 PR로 반영
```

hotfix 내용은 main에 자동 반영되지 않으므로, 다음 main 배포 시 hotfix가 누락되지 않도록 수동으로 main에도 반영해야 한다.

> 📍 관련 개념: `.study/00-dev-concepts.md` — Git Plumbing vs Porcelain, 릴리스 트레인

## `trigger-dotcom-hotfix.yml` — dotcom 긴급 배포

main에 머지된 특정 PR을 `hotfixes` 브랜치에 cherry-pick하여 tldraw.com을 즉시 배포하는 워크플로우. 정규 릴리스를 기다리지 않고 긴급 수정을 프로덕션에 내보낼 때 사용한다.

> 📍 .github/workflows/trigger-dotcom-hotfix.yml
> 📍 internal/scripts/trigger-dotcom-hotfix.ts

### 트리거 조건

| 이벤트 | 조건 | 시나리오 |
|--------|------|----------|
| `push` to `main` | PR에 `dotcom-hotfix-please` 라벨이 있을 때 | 라벨을 미리 붙여두고 머지 |
| `pull_request` + `labeled` | `merged == true` && 라벨이 `dotcom-hotfix-please` | 이미 머지된 PR에 라벨을 뒤늦게 부착 |

두 경우 모두 워크플로우는 실행되지만, 스크립트 내부에서 라벨 유무를 체크하여 **라벨이 없으면 조기 종료**한다.

### 전체 흐름

```
1) huppy-bot GitHub App 토큰 발급
2) origin/hotfixes 브랜치로 checkout
3) 해당 PR의 머지 커밋을 cherry-pick → hotfix/dotcom-{PR번호} 브랜치 생성
4) GitHub API로 hotfix PR 자동 생성 (hotfix/dotcom-{PR번호} → hotfixes)
   - 원본 PR 정보(제목, 작성자, API changes 섹션) 포함
5) CI 체크 통과 대기 (5분 초기 대기 → 15초 간격 폴링, 최대 15분)
   - mergeable_state가 'clean'이면 → squash merge 자동 실행
   - 'unstable' 또는 'dirty'면 → 에러
6) hotfixes 브랜치 머지 → trigger-production-build.yml이 프로덕션 배포 트리거
7) 매 단계 Discord 알림
```

### CI 대기 메커니즘

hotfix PR이 생성되면 레포에 설정된 기존 CI 워크플로우들이 자동 실행된다. 개별 check run을 추적하지 않고 GitHub의 `mergeable_state` API를 폴링하여 종합 결과만 확인한다.

```ts
// 5분 대기 후 15초 간격으로 폴링
const prStatus = await getPrDetailsByNumber(octokit, prNumber)
if (prStatus.mergeable_state === 'clean') {
  // squash merge 실행
  await octokit.rest.pulls.merge({ merge_method: 'squash', ... })
}
```

> 📍 관련 개념: `.study/00-dev-concepts.md` — Required Checks와 `mergeable_state`

## `trigger-sdk-hotfix.yml` — SDK/docs 긴급 패치 릴리스

main에 머지된 특정 PR을 현재 최신 릴리스 브랜치에 cherry-pick하여 npm 패치 버전을 배포하는 워크플로우.

> 📍 .github/workflows/trigger-sdk-hotfix.yml
> 📍 internal/scripts/trigger-sdk-hotfix.ts

### 트리거 조건

dotcom-hotfix와 동일한 구조이며, 라벨이 두 가지:

| 라벨 | 용도 |
|------|------|
| `sdk-hotfix-please` | SDK 패키지 변경 포함 — 테스트 실행 후 배포 |
| `docs-hotfix-please` | docs만 변경 — SDK 변경이 없는지 검증 후 배포 |

### 전체 흐름

```
1) npm에서 현재 최신 tldraw 버전 조회 (예: 3.8.0)
2) 릴리스 브랜치 결정 (예: v3.8.x)
3) 릴리스 브랜치로 checkout → cherry-pick
4) 분기:
   - docs-hotfix: 패키지 빌드 후 diff 비교 → SDK 변경이 있으면 거부
   - sdk-hotfix: 공개 패키지 대상 yarn test 전체 실행
5) 테스트/검증 통과 시 릴리스 브랜치에 직접 push → 패치 릴리스 파이프라인 트리거
6) 매 단계 Discord 알림
```

### dotcom-hotfix와의 차이

| | dotcom-hotfix | sdk-hotfix |
|---|---|---|
| **대상 브랜치** | `hotfixes` (고정) | `v{major}.{minor}.x` (npm 최신 버전 기반 동적 결정) |
| **배포 방식** | PR 생성 → CI 대기 → 자동 머지 | 테스트 직접 실행 → 릴리스 브랜치에 바로 push |
| **배포 대상** | tldraw.com 웹앱 | npm SDK 패키지 / docs 사이트 |
| **안전장치** | GitHub required checks (외부 CI) | 자체 테스트 실행 또는 SDK diff 검증 |

dotcom은 중간 PR을 만들어 기존 CI 인프라를 활용하고, SDK는 스크립트 내에서 직접 테스트를 돌린다.

> 📍 관련 개념: `.study/00-dev-concepts.md` — GitHub App 권한 체계, 릴리스 트레인

## `dependabot-dedupe.yml` — Dependabot PR 자동 중복 제거

Dependabot이 만든 PR 브랜치에서 `yarn dedupe`를 자동 실행하여 lock 파일 중복을 정리하는 워크플로우.

> 📍 .github/workflows/dependabot-dedupe.yml

### 트리거

`dependabot/**` 브랜치에 push 시 (= Dependabot이 PR을 만들 때).

### 동작

```
1) checkout
2) yarn dedupe 실행
3) 변경이 있으면 커밋/푸시 (변경 없으면 스킵)
```

Dependabot은 하나의 패키지만 업데이트하고 나머지 lock 파일 엔트리는 건드리지 않으므로, 기존에 통합되어 있던 버전이 다시 중복될 수 있다. 이 워크플로우가 그 뒤처리를 자동으로 해준다.

> 📍 관련 개념: `.study/00-dev-concepts.md` — Dependabot, yarn dedupe, Dependabot이 중복을 악화시키는 패턴

## `issue-triage.yml` — Claude AI 이슈 자동 분류

새 이슈가 생성되면 Claude AI가 자동으로 제목 정리, 번역, 타입 분류, 라벨링, 마일스톤 배정을 수행하는 워크플로우.

> 📍 .github/workflows/issue-triage.yml

### 트리거

- 이슈 생성(`issues: opened`) 시 자동 실행
- 수동 실행(`workflow_dispatch`) — issue_number 입력하여 테스트 가능

### 동작 (2단계)

**Step 1 — 이슈 정보 수집:**

```bash
# GitHub CLI로 이슈 메타데이터를 JSON으로 수집
gh issue view 123 --json title,body,labels,state > /tmp/issue.json

# 이슈 타입은 gh CLI가 아직 미지원이라 API로 별도 조회
gh api repos/tldraw/tldraw/issues/123 --jq '.type.name // "none"'

# 둘을 jq로 합침
```

**Step 2 — Claude에게 5가지 작업 지시:**

`anthropics/claude-code-action@v1`을 사용하여 Claude Code를 GitHub Actions 안에서 실행:

| # | 작업 | 예시 |
|---|------|------|
| 1 | 제목 개선 | sentence case, 버그면 증상 기술, 기능이면 명령형 |
| 2 | 본문 개선 | 영어 외 언어 → 영문 번역 추가 (원문 유지) |
| 3 | 이슈 타입 | Bug / Feature / Task / Example 분류 |
| 4 | 라벨 적용 | `sdk`, `dotcom`, `performance`, `bugfix` 등 |
| 5 | 마일스톤 배정 | 해당하면 자동 배정 (대부분은 미배정) |

**타입 분류 기준 — Bug vs Feature 구분이 핵심:**

- Bug: "클릭하면 크래시 발생", "텍스트가 렌더링 안 됨" (실제 고장)
- Feature: "버튼 색이 마음에 안 든다", "메뉴가 왼쪽에 있으면 좋겠다" (UX 선호)

### 보안 제한

```yaml
claude_args: '--allowedTools "Bash(gh issue:*),Bash(gh api:*)"'
```

Claude가 실행할 수 있는 명령어를 `gh issue`와 `gh api`로만 제한. 파일 수정, 임의 셸 명령 실행 불가. 이슈에 댓글도 달지 않음(조용히 메타데이터만 수정).

### `anthropics/claude-code-action@v1`

Anthropic이 만든 공식 GitHub Action. GitHub Actions runner에서 Claude Code CLI를 실행하여, Claude가 **실제로 셸 명령어를 실행**할 수 있게 해준다.

| | Claude API 직접 호출 | claude-code-action |
|---|---|---|
| 할 수 있는 것 | 텍스트 응답만 | 셸 명령 실행, 파일 읽기/쓰기 등 |
| 본질 | 채팅 API | Claude Code (에이전트) |

`--allowedTools` 옵션으로 허용 명령어를 제한할 수 있어, 자동화에서 안전하게 사용 가능.
