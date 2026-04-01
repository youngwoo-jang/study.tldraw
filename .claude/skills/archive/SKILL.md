---
name: archive
skill-manager-id: eric/skills/archive-6b0bd37d
skill-manager-writer: eric
skill-manager-updated: 2026-03-22T04:15:37.074Z
---

# Archive

사용자가 tldraw 오픈소스 분석 중 발견한 레슨들을 `.study/` 디렉토리의 카테고리별 마크다운 파일에 정리한다.

## 카테고리 & 파일 매핑

| # | 카테고리 | 파일 | 키워드 |
|---|---------|------|--------|
| 0 | 개발 개념 & 용어 | `00-dev-concepts.md` | 빌드, 번들링, 트랜스파일, 증분 빌드, 캐싱, 모노레포 오케스트레이터, CJS, ESM, 트리 쉐이킹, 코드 스플리팅, 소스맵, HMR, 타입 체크, 린팅, 포매팅 |
| 1 | 개발 인프라 & 설정 | `01-dev-infra.md` | vitest, eslint, oxlint, yarn berry, lazyrepo, prettier, CI/CD, GitHub Actions, Sentry, 에러 핸들링, DX, 예제 앱, VSCode 확장, 템플릿, CLI, 빌드 캐싱 |
| 2 | UI 아키텍처 | `02-ui-architecture.md` | 컴포넌트 구조, 반응형 UI, 컴포넌트 오버라이드 시스템 |
| 3 | 아키텍처 패턴 | `03-architecture-patterns.md` | 모노레포, 패키지 의존성, 레이어드 설계, 확장 시스템 |
| 4 | 디자인 패턴 | `04-design-patterns.md` | ShapeUtil, BindingUtil, 전략 패턴, 팩토리 패턴 |
| 5 | 외부 인터페이스 & API 설계 | `05-api-design.md` | 공개 API, 모듈 경계, 타입 시스템, 스키마, 마이그레이션, API Extractor |
| 6 | 상태 관리 | `06-state-management.md` | 리액티브 시그널, Atom, Computed, 상태 머신, StateNode, 이벤트 드리븐 |
| 7 | 성능 최적화 | `07-performance.md` | 리렌더 방지, 렌더링 최적화, 메모이제이션, 번들 사이즈 |
| 8 | 테스트 전략 | `08-testing.md` | 유닛 테스트, E2E, TestEditor, Playwright, 테스트 설계 |
| 9 | 리소스 & 에셋 관리 | `09-resource-management.md` | 폰트, 아이콘, 번역, i18n, 에셋 번들링, CDN, refresh-assets |
| 10 | 캔버스 & 렌더링 | `10-canvas-rendering.md` | Canvas, SVG, 좌표계, 변환, 히트 테스트, 줌, 패닝, 카메라, 지오메트리 |
| 11 | 멀티플레이어 & 동기화 | `11-multiplayer-sync.md` | CRDT, 동기화, 클라이언트-서버, 오프라인, sync-worker |

## 워크플로우

1. 사용자가 레슨/인사이트를 던지면, 대화 전체를 분석하여 아카이빙할 내용을 추출한다.
2. `.study/{파일명}`이 없으면 새로 생성하고, 있으면 기존 내용에 병합한다.
3. **멀티 카테고리 분배** (중요): 하나의 대화에서 여러 카테고리에 해당하는 내용이 나올 수 있다. 반드시 모든 관련 카테고리를 식별하고 각각에 적절한 수준으로 작성한다.
4. 어떤 카테고리에도 맞지 않으면 사용자에게 새 카테고리 추가 여부를 확인한다.

### 멀티 카테고리 분배 규칙

대화에서 발견된 내용을 세 계층으로 분류하여 각 카테고리에 적절히 배분한다:

- **범용 개념** → `00-dev-concepts.md`: 특정 프로젝트에 국한되지 않는 일반적 개념/용어 정의 (예: "단위테스트 vs 통합테스트란 무엇인가")
- **인프라/도구 설정** → `01-dev-infra.md` 등: 프로젝트에서 어떤 도구를 쓰고 어떻게 설정했는지 요약 + 상세 카테고리로 크로스 레퍼런스 (예: "Vitest를 쓴다, 설정은 이렇다 → 상세는 08-testing.md")
- **주제별 상세** → 해당 카테고리: 깊이 있는 분석과 코드 스니펫 (예: `08-testing.md`에 프리셋 분석, 폴리필 상세, 워크스페이스별 차이)

예시 — "Vitest 테스트 설정" 주제라면:
1. `00-dev-concepts.md` ← 단위/통합/E2E 테스트의 일반적 정의
2. `01-dev-infra.md` ← "Vitest를 쓴다 + 설정 구조 요약 + `> 📍 상세: .study/08-testing.md`"
3. `08-testing.md` ← 프리셋 분석, setup.ts 상세, 워크스페이스별 차이 등 전체 내용

## 레슨 작성 규칙

- 각 레슨은 `##` 제목으로 시작한다.
- 제목 아래 한 줄 요약을 둔다.
- 관련 소스 코드 경로를 `> 📍 파일경로:라인번호` 형태로 표기한다.
- 코드 스니펫은 핵심 부분만 간결하게 포함한다.
- 같은 파일 내 레슨 간 중복이 생기면 병합한다.

## 레슨 파일 기본 구조

```markdown
# {카테고리명}

## {레슨 제목}

{한 줄 요약}

> 📍 packages/editor/src/lib/Editor.ts:123

{본문 설명}

```ts
// 핵심 코드 스니펫
```
```
