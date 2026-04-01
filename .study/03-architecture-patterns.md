# 아키텍처 패턴

## 모듈 분리 단계 — 폴더 → 모노레포 → npm

코드 모듈화는 점진적으로 진행한다. 각 단계는 이전 단계의 한계가 체감될 때 넘어간다.

### 1단계: 폴더 분리

```
src/
├── utils/
├── auth/
└── api/
```

- 같은 빌드, 같은 배포 단위
- import 경로만 다름
- **넘어가는 시점**: 빌드가 느려지거나, 팀/도메인 경계가 명확해질 때

### 2단계: 모노레포 패키지 분리

```
packages/
├── utils/    ← 독립 package.json
├── auth/
└── api/
```

- 각 패키지가 독립 빌드/테스트 가능
- 같은 레포 안에서 심볼릭 링크로 참조 (yarn workspaces 등)
- 코드 바꾸면 바로 반영, 디버깅 편리
- **넘어가는 시점**: 다른 프로젝트/팀에서도 이 패키지를 쓰고 싶을 때

### 3단계: npm 패키지 배포

```bash
npm install @my-org/utils
```

- 독립 버전 관리, 독립 릴리스 사이클
- 어떤 프로젝트든 npm install로 가져다 씀
- Lerna 같은 도구로 버전/퍼블리시 자동화

### npm 분리의 비용

모노레포 패키지 대비 추가되는 오버헤드:

| | 모노레포 패키지 | npm 패키지 |
|---|---|---|
| 버전 관리 | 불필요 | 필수 (semver) |
| 코드 반영 | 즉시 | 배포 → 설치 사이클 |
| 디버깅 | 같은 레포 안 | node_modules 안 |
| 의존성 관리 | 자동 (워크스페이스) | 명시적 (버전 범위) |

**원칙: 모노레포 패키지로 충분하면 npm 분리하지 않는다.**

tldraw는 내부 개발은 모노레포로 하되, SDK를 외부 사용자에게 제공하기 위해 npm에 배포한다. 즉 npm 배포의 동기는 "내부 분리"가 아니라 "외부 제공".

> 📍 관련: `.study/00-dev-concepts.md` — npm 배포, Lerna, 모듈 분리 전략
> 📍 관련: `.study/01-dev-infra.md` — tldraw의 Lerna + 커스텀 퍼블리싱 인프라

## 레이어드 SDK 패키지 설계

tldraw는 코어 엔진과 UI를 별도 패키지로 분리하여, 소비자가 원하는 수준에서 사용할 수 있는 레이어 구조를 제공한다.

```
packages/editor     →  코어 엔진 (캔버스, Editor 클래스, 상태머신, 도형 시스템)
                        UI 없음. "헤드리스" 에디터.
packages/tldraw     →  editor + 기본 UI + 기본 도형/툴 전부 묶은 "batteries included" SDK
                        대부분의 소비자는 이것만 쓰면 됨.
```

### 소비 티어

| 티어 | 진입점 | 사용 사례 |
|------|--------|-----------|
| 1 | `<Tldraw />` (packages/tldraw) | 한 줄로 에디터 전체 임베딩. 90% 사용 사례 |
| 2 | `<TldrawEditor />` + 커스텀 UI (packages/editor) | UI를 직접 구성하고 싶을 때 |
| 3 | `Editor` 클래스 직접 사용 (packages/editor) | 프로그래밍 방식으로 도형 조작, 커스텀 상태머신 |

### re-export 패턴

`packages/tldraw/src/index.ts`에서 `export * from '@tldraw/editor'`로 하위 패키지를 통째로 re-export한다. 소비자는 `tldraw` 하나만 import하면 editor의 모든 API도 함께 사용할 수 있다. 200개 이상의 export가 단일 패키지 경로로 제공된다.

```tsx
// 소비자는 tldraw 하나에서 모든 것을 import
import { Tldraw, Editor, useEditor, ShapeUtil } from 'tldraw'
```

> 📍 packages/tldraw/src/index.ts:56 — `export * from '@tldraw/editor'`
> 📍 관련: `.study/02-ui-architecture.md` — Tldraw 컴포넌트 합성 구조
> 📍 관련: `.study/05-api-design.md` — Progressive disclosure API 패턴
