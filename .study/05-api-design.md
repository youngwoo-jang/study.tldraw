# 외부 인터페이스 & API 설계

## TSDoc + API Extractor 파이프라인

TSDoc 주석과 API Extractor를 조합하여 공개 API를 정의하고, CI에서 자동 검증 및 문서 생성까지 연결하는 파이프라인.

> 📍 tsdoc.json
> 📍 internal/config/api-extractor.json
> 📍 internal/scripts/build-api.ts
> 📍 internal/scripts/api-check.ts

### tsdoc.json — TSDoc 설정

```json
{
  "extends": ["@microsoft/api-extractor/extends/tsdoc-base.json"],
  "tagDefinitions": [{ "tagName": "@react", "syntaxKind": "modifier" }],
  "supportForTags": { "@react": true }
}
```

- API Extractor의 기본 TSDoc 규칙을 확장
- 커스텀 `@react` modifier 태그 정의 — React 전용 API(컴포넌트/훅)를 표시하는 용도
- 실제 사용 예: `/** @public @react */` (TldrawEditor.tsx 등)

> 📍 관련 개념: `.study/00-dev-concepts.md` — TSDoc, API Extractor

### API Extractor 설정 구조 (상속 패턴)

```
internal/config/api-extractor.json   ← 베이스 설정 (모든 옵션 정의)
    ↑ extends
packages/editor/api-extractor.json   ← 한 줄 ("extends": "../../internal/config/api-extractor.json")
packages/tldraw/api-extractor.json   ← 동일
packages/store/api-extractor.json    ← 동일
... (15개 패키지)
```

토큰 기반 경로로 패키지별 자동 적용:
- `<unscopedPackageName>` → 패키지 이름 (e.g., `editor`, `tldraw`)
- `<projectFolder>` → `../../` (모노레포 루트)

### 진입점과 산출물

**진입점**: 각 패키지의 `index.d.ts` 하나만 본다.

```json
"mainEntryPointFilePath": "<projectFolder>/packages/<unscopedPackageName>/.tsbuild-api/index.d.ts"
```

API Extractor는 이 진입점에서 export 체인만 따라간다. `index.ts`에서 export하지 않은 모듈은 존재 자체를 모른다. 즉 **API 관리의 첫 번째 게이트는 `index.ts`에 export를 추가하는 것** 자체.

**산출물 4가지와 각 용도**:

| 산출물 | 설정 키 | 용도 |
|--------|---------|------|
| `api/public.d.ts` | `publicTrimmedFilePath` | `@public`만 모은 .d.ts 롤업. api-check에서 소비자 관점 타입 검증용 |
| `api/internal.d.ts` | `untrimmedFilePath` | 전체 export 포함 .d.ts 롤업. 개발자 참조용 |
| `api/api.json` | `apiJsonFilePath` | 구조화된 문서 모델. docs 사이트 API 레퍼런스 자동 생성용 |
| `api-report.api.md` | `apiReport` | git에 커밋되는 API 목록. PR에서 변경 감지, CI에서 불일치 시 빌드 실패 |

### build-api 실행 과정

> 📍 internal/scripts/build-api.ts

```
1. .tsbuild/ → .tsbuild-api/ 복사 (원본 캐시 보호)
2. sortUnions — .tsbuild-api/ 내 모든 .d.ts의 union 멤버를 알파벳순 정렬
3. api/ 디렉토리 초기화 (rimraf)
4. api-extractor run 실행
```

**Step 1 — 복사하는 이유**: 다음 단계에서 `.d.ts`를 수정(union 정렬)해야 하는데, 원본 `.tsbuild`를 건드리면 lazyrepo 캐시가 깨진다.

**Step 2 — sortUnions가 필요한 이유**: tsc는 union 멤버 순서를 비결정적으로 출력한다. 코드 변경 없이 빌드만 다시 해도 순서가 달라질 수 있어서, 알파벳순 정렬로 항상 동일한 출력을 보장한다.

> 📍 internal/scripts/lib/sort-unions.ts

```ts
// recast AST 파서로 두 가지를 정렬
visit(code, {
  visitTSUnionType(path) {    // type A = C | A | B → A | B | C
    val.types = val.types.sort(...)
  },
  visitTSTypeLiteral(path) {  // { c: 1; a: 2 } → { a: 2; c: 1 }
    val.members = val.members.sort(...)
  },
})
```

**Step 4 — CI vs 로컬 동작 차이**:

```ts
await exec('yarn', ['run', '-T', 'api-extractor', 'run', isCI ? null : '--local'])
```

- CI: `api-extractor run` — api-report.api.md가 달라지면 에러. 빌드 실패
- 로컬: `api-extractor run --local` — api-report.api.md를 자동 업데이트. 개발자가 커밋

실패 시 기존 리포트와 새 리포트의 git diff를 출력하여 뭐가 달라졌는지 보여준다.

### api-check — 소비자 관점 타입 검증

> 📍 internal/scripts/api-check.ts

`api/public.d.ts`만으로 타입이 정합한지 검증하는 "API set validation".

```
1. 모든 패키지의 api/public.d.ts를 임시 디렉토리에 복사
2. 패키지 간 import를 paths로 연결하는 tsconfig 생성
3. 그것만으로 tsc 실행
```

```ts
// 임시 디렉토리에 이런 구조를 만듦
editor.d.ts        ← packages/editor/api/public.d.ts 복사
tldraw.d.ts        ← packages/tldraw/api/public.d.ts 복사
tsconfig.json      ← paths: { "@tldraw/editor": ["./editor.d.ts"] }
```

잡아내는 실수들:
- `@public` 함수가 `@internal` 타입을 파라미터로 쓰는 경우
- export는 했는데 의존하는 타입의 패키지를 빠뜨린 경우
- public API끼리의 타입 불일치

npm에 배포한 뒤 사용자가 "타입 에러 나요"라고 하기 전에, CI에서 미리 소비자 환경을 시뮬레이션하는 것.

### 에러 정책

베이스 설정에서 모든 메시지 레벨을 `error`로 설정 (엄격):

- 컴파일러/API Extractor/TSDoc 메시지: 전부 `error`
- `ae-internal-missing-underscore` → `none` (내부 API에 `_` 접두사 강제하지 않음)
- `ae-wrong-input-file-type` → `none`
- `ae-forgotten-export` → `error` (export 빠뜨리면 빌드 실패)

### docs 사이트 연동

`api/api.json`을 `apps/docs`에서 `@microsoft/api-extractor-model`로 읽어 API 레퍼런스 페이지를 자동 생성한다.

```ts
// lazy.config.ts — docs 빌드는 모든 패키지의 build-api 완료 후 실행
'apps/docs': {
  runsAfter: { 'build-api': { in: 'all-packages' } },
}
```

`api.json`의 `projectFolderUrl`이 `https://github.com/tldraw/tldraw/blob/main`으로 설정되어 GitHub 소스 링크도 자동 생성.

### 전체 파이프라인 요약

이 시스템은 개발 중 import를 강제하거나 리다이렉트하는 것이 아니라, **사람의 실수를 CI가 자동으로 잡아주는 안전망**이다:

- `@public` 붙여야 할 걸 빠뜨리거나
- `@internal`인 타입을 public API에서 실수로 노출하거나
- public API 시그니처를 본인도 모르게 바꾸거나

이런 경우 CI에서 빌드가 실패하여 머지를 차단한다.

```
소스 (.ts)
  ↓ tsc (build-types) — src 내 모든 .ts 컴파일
.tsbuild/*.d.ts + .js
  ↓ cp -R
.tsbuild-api/*.d.ts (사본)
  ↓ sortUnions (빌드 안정성)
.tsbuild-api/*.d.ts (정렬됨)
  ↓ api-extractor run — index.d.ts 진입점에서 export 체인만 추적
  ├── api/public.d.ts        → api-check 검증
  ├── api/internal.d.ts      → 참조용
  ├── api/api.json           → docs 사이트 API 레퍼런스
  └── api-report.api.md      → PR 리뷰 시 API 변경 추적
  ↓ api-check
  임시 프로젝트에서 public.d.ts만으로 tsc → 소비자 관점 타입 정합성 보장
```
