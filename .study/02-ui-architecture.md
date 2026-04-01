# UI 아키텍처

## Tldraw 컴포넌트 합성 구조

SDK의 메인 진입점인 `<Tldraw />` 컴포넌트는 React Context 기반의 Provider 체인으로 구성된다. 각 레이어가 독립적이어서 소비자가 원하는 수준에서 커스텀할 수 있다.

> 📍 packages/tldraw/src/lib/Tldraw.tsx

```tsx
function Tldraw(props) {
  return (
    <AssetUrlsProvider>              {/* 에셋 URL 관리 */}
      <TldrawUiTranslationProvider>  {/* i18n */}
        <TldrawEditor>               {/* 코어 엔진: Editor 인스턴스 생성 */}
          <TldrawUi>                 {/* UI 레이어: 툴바, 메뉴, 다이얼로그 */}
            {children}
          </TldrawUi>
        </TldrawEditor>
      </TldrawUiTranslationProvider>
    </AssetUrlsProvider>
  )
}
```

### 레이어별 역할

| 레이어 | 패키지 | 역할 |
|--------|--------|------|
| `TldrawEditor` | `@tldraw/editor` | `Editor` 인스턴스 생성 → `EditorProvider`로 Context에 주입 |
| `TldrawUi` | `@tldraw/tldraw` | `useEditor()` hook으로 Editor를 소비하며 전체 UI 렌더링 |
| `Tldraw` | `@tldraw/tldraw` | 위 두 레이어 + 기본 도형/툴/에셋을 묶은 "batteries included" 래퍼 |

### 기본값 병합 패턴

`Tldraw` 컴포넌트는 사용자가 전달한 props를 기본값과 병합한다. 병합 방식은 props 종류에 따라 다르다.

**배열 props (shapeUtils, tools, bindingUtils) — 키 기반 교체**

> 📍 packages/utils/src/lib/array.ts — `mergeArraysAndReplaceDefaults`
> 📍 packages/tldraw/src/lib/Tldraw.tsx:161-176

```ts
function mergeArraysAndReplaceDefaults(key, customEntries, defaults) {
  const overrideTypes = new Set(customEntries.map(entry => entry[key]))
  const result = []
  for (const defaultEntry of defaults) {
    if (overrideTypes.has(defaultEntry[key])) continue  // 같은 키면 기본값 스킵
    result.push(defaultEntry)
  }
  for (const customEntry of customEntries) {
    result.push(customEntry)  // 커스텀은 항상 추가
  }
  return result
}

// shapeUtils는 'type' 키로, tools는 'id' 키로 매칭
mergeArraysAndReplaceDefaults('type', userShapeUtils, defaultShapeUtils)
mergeArraysAndReplaceDefaults('id', userTools, allDefaultTools)
```

같은 `type`/`id`면 사용자 것이 기본값을 교체하고, 새로운 것이면 기본값에 추가된다. 기본 12개 ShapeUtil 중 하나만 교체하고 나머지는 유지할 수 있다.

**객체 props (components) — 스프레드 병합**

```ts
const componentsWithDefault = {
  Scribble: TldrawScribble,
  SelectionForeground: TldrawSelectionForeground,
  // ... 기본 컴포넌트들
  ..._components,  // 사용자 것이 뒤에서 덮어씀
}
```

**객체 props (options) — 스프레드 + 부분 깊은 병합**

```ts
const optionsWithDefaults = { ...options, text: textOptionsWithDefaults }
// text 옵션만 한 단계 더 깊이 병합
```

### Editor 인스턴스와 React의 관계

`Editor` 클래스는 순수 TypeScript 객체로, React 밖에서도 동작할 수 있다. React는 이 객체를 생성하고 Context로 전달하는 "호스팅" 역할만 한다. Context는 **편의 배포 수단**이지, Editor의 생명주기를 소유하는 게 아니다.

```
Editor (순수 TS 클래스)
  ├── HistoryManager    — 실행취소/다시실행
  ├── SnapManager       — 스냅핑
  ├── TextManager       — 텍스트 측정
  ├── InputsManager     — 포인터 추적
  ├── FontManager       — 폰트 로딩
  └── ... (7개+ 매니저)
```

> 📍 packages/editor/src/lib/editor/Editor.ts
> 📍 packages/editor/src/lib/TldrawEditor.tsx

UI 컴포넌트들은 `useEditor()` hook으로 이 인스턴스에 접근하여 상태를 읽고 명령을 보낸다. Editor의 상태는 리액티브 시그널 기반이므로, 상태가 변하면 관련 컴포넌트만 자동으로 리렌더된다.

### React 외부에서 Editor 접근하기

Context 안에서만 접근할 수 있다면 외부 연동이 어려워진다. tldraw는 3가지 탈출구를 제공한다:

> 📍 packages/editor/src/lib/hooks/useEditor.tsx
> 📍 packages/tldraw/src/lib/Tldraw.tsx:108-253

**1. `onMount` 콜백 — 같은 인스턴스의 참조를 꺼냄 (가장 일반적)**

```tsx
const editorRef = useRef<Editor | null>(null)

<Tldraw onMount={(editor) => {
  editorRef.current = editor  // 별도 생성이 아니라 참조 공유
}} />

// 이후 editorRef.current로 외부에서 동일 인스턴스 접근
```

**2. `new Editor()` 직접 생성 — React 없이 단독 사용 (테스트, 서버사이드, headless)**

```ts
const editor = new Editor({
  store: createTLStore({ shapeUtils: [...] }),
  getContainer: () => document.body,
  shapeUtils: [...], tools: [], bindingUtils: [],
})
```

별도 인스턴스이므로 React 트리의 Editor와는 sync되지 않는다. React 앱에서 외부 접근이 필요하면 `onMount`를 써야 한다.

**3. `useMaybeEditor()` — nullable 버전**

```ts
const editor = useMaybeEditor()  // Editor | null
```

Context 밖에서 호출해도 에러 대신 `null`을 반환한다.

> 📍 관련: `.study/06-state-management.md` — 리액티브 시그널, 상태 머신

---

## TldrawUi 내부 Provider 체인

`TldrawUi`는 내부적으로 9개 이상의 Context Provider를 중첩하여 UI 전역 상태를 관리한다.

> 📍 packages/tldraw/src/lib/ui/context/TldrawUiContextProvider.tsx:61-100

```
TldrawUiContextProvider (최상위)
├── MimeTypeContext.Provider        — 지원 MIME 타입
│   └── AssetUrlsProvider           — 아이콘/에셋 URL
│       └── TldrawUiTranslationProvider — i18n 번역
│           └── TldrawUiTooltipProvider — 툴팁 관리
│               └── TldrawUiEventsProvider — UI 이벤트 추적
│                   └── TldrawUiToastsProvider — 토스트 알림
│                       └── TldrawUiDialogsProvider — 다이얼로그/모달
│                           └── TldrawUiA11yProvider — 접근성 (스크린 리더)
│                               └── BreakPointProvider — 반응형 브레이크포인트
│                                   └── TldrawUiComponentsProvider — UI 컴포넌트 오버라이드
│                                       └── InternalProviders
│                                           ├── ActionsProvider — 100+ UI 액션
│                                           └── ToolsProvider — 15+ 도구 정의
```

### 주요 Context 요약

| 시스템 | Hook | 역할 |
|--------|------|------|
| Actions | `useActions()` | undo, redo, align, group, export 등 100+ 액션 |
| Tools | `useTools()` | select, hand, eraser, draw, geo 등 15+ 도구 |
| Dialogs | `useDialogs()` | `addDialog()` / `removeDialog()` — Radix UI Dialog 기반 |
| Toasts | `useToasts()` | `addToast()` — severity별 알림, 4초 자동 닫힘 |
| Events | `useUiEvents()` | 50+ 이벤트 타입, 이벤트 소스(menu/toolbar/kbd) 추적 |
| Breakpoints | `useBreakpoint()` | 뷰포트 너비 기반 반응형 단계 |
| A11y | `useA11y()` | 스크린 리더용 `polite`/`assertive` 안내 메시지 |

---

## UI 레이아웃 구조

`TldrawUiContent`가 실제 UI 배치를 담당한다. 모든 영역의 컴포넌트는 오버라이드 가능하다.

> 📍 packages/tldraw/src/lib/ui/TldrawUi.tsx:97-236

```
tlui-layout (루트)
├── tlui-layout__top
│   ├── __top__left       → MenuPanel, HelperButtons
│   ├── __top__center     → TopPanel
│   └── __top__right      → SharePanel, StylePanel
├── tlui-layout__bottom
│   ├── __bottom__main    → NavigationPanel, Toolbar, HelpMenu
│   └── DebugPanel        (디버그 모드 전용)
├── Toasts                (전역 레이어)
├── Dialogs               (전역 레이어)
└── A11y                  (접근성 레이어)
```

포커스 모드에서는 전체 UI가 숨겨지고 토글 버튼만 남는다.

---

## 컴포넌트 오버라이드 시스템

tldraw의 핵심 확장 메커니즘. 소비자가 기본 UI 컴포넌트를 자신만의 구현으로 교체하거나 `null`로 비활성화할 수 있다.

### 두 계층의 오버라이드

**에디터 레벨** (`TLEditorComponents`) — 캔버스 관련 27개 컴포넌트:

> 📍 packages/editor/src/lib/hooks/EditorComponentsContext.tsx:20-52

```
Background, Brush, Canvas, Cursor, Grid, Handle,
InFrontOfTheCanvas, OnTheCanvas, SelectionBackground,
SelectionForeground, ShapeIndicator, SnapIndicator,
CollaboratorCursor, CollaboratorHint, Spinner, SvgDefs ...
```

**UI 레벨** (`TLUiComponents`) — 인터페이스 관련 25개 컴포넌트:

> 📍 packages/tldraw/src/lib/ui/context/components.tsx:49-76

```
ContextMenu, MainMenu, Toolbar, StylePanel, Minimap,
NavigationPanel, PageMenu, TopPanel, SharePanel,
KeyboardShortcutsDialog, Dialogs, Toasts, A11y ...
```

**통합 타입** — `<Tldraw />`에서는 두 계층을 합친 단일 prop으로 제공:

> 📍 packages/tldraw/src/lib/Tldraw.tsx:72

```ts
export interface TLComponents extends TLEditorComponents, TLUiComponents {}
```

### 오버라이드 병합 메커니즘

기본값과 사용자 오버라이드를 **spread 연산자**로 병합한다. 사용자 오버라이드가 뒤에 오므로 기본값을 덮어쓴다.

> 📍 packages/editor/src/lib/hooks/useEditorComponents.tsx:39-75

```tsx
const value = useMemo(
  (): Required<TLEditorComponents> => ({
    Background: DefaultBackground,
    Brush: DefaultBrush,
    Canvas: DefaultCanvas,
    // ... 20+ 기본값 ...
    ..._overrides,  // 사용자 오버라이드가 기본값을 덮어씀
  }),
  [_overrides]
)
```

**최적화**: `useShallowObjectIdentity()`로 overrides 객체를 안정화하여 불필요한 리렌더를 방지한다. SDK 소비자가 인라인 객체 `components={{ Toolbar: MyToolbar }}`를 넘기면 부모 리렌더마다 새 참조가 생기는데, shallow equality로 내용이 같으면 이전 참조를 유지하여 `useMemo`가 재계산되지 않게 한다. 소비자에게 `useMemo`를 강제할 수 없으므로 라이브러리 내부에서 방어적으로 처리하는 패턴.

> 📍 packages/editor/src/lib/hooks/useIdentity.tsx

### Context = IoC 컨테이너

컴포넌트를 Context에 넣는 이유는 **간접 참조(indirection)**를 만들기 위해서다. SDK 내부 코드가 `DefaultBrush`를 직접 import하면, 소비자가 교체할 방법이 없다.

```tsx
// ❌ 직접 import — 하드코딩, 교체 불가
import { DefaultBrush } from './DefaultBrush'
return <DefaultBrush brush={brush} />

// ✅ Context 경유 — 소비자가 주입한 컴포넌트로 교체됨
const { Brush } = useEditorComponents()
return <Brush brush={brush} />
```

Context가 추상과 구현 사이의 접합점 역할을 한다. SDK 내부는 "Brush가 뭔지" 모르고 `useEditorComponents().Brush`만 알며, 실제로 뭐가 들어올지는 SDK 소비자가 결정한다. 전형적인 IoC(Inversion of Control) 패턴을 React Context로 구현한 것.

**안티패턴 아닌 이유**: "컴포넌트를 Context/state에 넣지 말라"는 조언은 **React Element** (`<Comp />` = `createElement()` 반환값)를 저장하는 경우에 해당한다. Element는 props가 생성 시점에 고정되어 stale해진다. 반면 tldraw가 저장하는 것은 **Component Type** (함수 참조)이므로, 렌더링 시점에 최신 props를 전달할 수 있다.

| | Element (`<Comp />`) | Component Type (`Comp`) |
|---|---|---|
| 정체 | `createElement()` 결과 객체 | 함수 자체 |
| props | 생성 시점에 고정 (stale 위험) | 사용 시점에 전달 (항상 최신) |

### 소비 패턴

컴포넌트를 렌더링할 때 hook으로 꺼내서 조건부 렌더링한다:

```tsx
// 에디터 레벨 — DefaultCanvas 내부
const { Background, InFrontOfTheCanvas } = useEditorComponents()

return (
  <>
    {Background && <Background />}
    {InFrontOfTheCanvas && <InFrontOfTheCanvas />}
  </>
)
```

```tsx
// UI 레벨 — TldrawUiContent 내부
const { MenuPanel, Toolbar, Dialogs, Toasts } = useTldrawUiComponents()

return (
  <div className="tlui-layout">
    {MenuPanel && <MenuPanel />}
    {Toolbar && <Toolbar />}
    {Dialogs && <Dialogs />}
    {Toasts && <Toasts />}
  </div>
)
```

### `null`로 비활성화

컴포넌트를 `null`로 설정하면 해당 영역이 완전히 사라진다:

```tsx
<Tldraw components={{ Minimap: null, HelpMenu: null }} />
```

### 훅 기반 오버라이드 (Actions / Tools)

컴포넌트 교체와 달리, 액션과 도구는 **함수 체이닝**으로 오버라이드한다:

> 📍 packages/tldraw/src/lib/ui/overrides.ts:152-187

```ts
export function mergeOverrides(
  overrides: TLUiOverrides[],
  defaultHelpers: TLUiOverrideHelpers
): TLUiOverridesWithoutDefaults {
  return {
    actions: (editor, schema, helpers) => {
      for (const override of overrides) {
        if (override.actions) {
          schema = override.actions(editor, schema, helpers) // 체인 변환
        }
      }
      return schema
    },
    tools: (editor, schema, helpers) => { /* 동일 패턴 */ }
  }
}
```

이전 결과를 다음 오버라이드 함수의 입력으로 전달하여, 여러 오버라이드를 순차적으로 적용할 수 있다.

---

## 반응형 UI

### useValue — 시그널 구독 hook

Editor의 리액티브 시그널을 React 컴포넌트에 연결하는 핵심 hook. TypeScript 함수 오버로드로 두 가지 사용법을 제공한다.

> 📍 packages/state-react/src/lib/useValue.ts

```tsx
// 오버로드 1: 시그널 직접 구독
const value = useValue(someSignal)

// 오버로드 2: computed 생성 + 구독 (더 일반적)
const pages = useValue('pages', () => editor.getPages(), [editor])
const currentPage = useValue('currentPage', () => editor.getCurrentPage(), [editor])
```

오버로드 2의 `name` 인자는 디버깅용이다. `whyAmIRunning()` 등에서 `Computed(pages) is recomputing because: ...` 형태로 출력된다.

**내부 동작**:
1. `computed()`로 리액티브 값 생성 (오버로드 2) 또는 시그널 직접 사용 (오버로드 1)
2. `react()`로 시그널 변경 감지 → `notify()` 콜백 호출
3. React의 `useSyncExternalStore`로 리렌더 트리거 — concurrent mode에서 tearing 방지
4. `lastChangedEpoch`(정수)로 변경 여부 판단
5. `__unsafe__getWithoutCapture()`로 값 반환 — 이미 subscribe에서 변경을 감지하고 있으므로 불필요한 의존성 추적을 건너뜀

```ts
const { subscribe, getSnapshot } = useMemo(() => ({
  subscribe: (notify) => {
    return react(`useValue(${name})`, () => {
      $val.get()   // 의존성 캡처
      notify()     // React 리렌더 트리거
    })
  },
  getSnapshot: () => $val.lastChangedEpoch,
}), deps)

useSyncExternalStore(subscribe, getSnapshot, getSnapshot)
return $val.__unsafe__getWithoutCapture()  // 추적 없이 값만 반환
```

> 📍 상세: `.study/06-state-management.md` — 시그널 시스템, epoch 기반 변경 감지
> 📍 관련: `.study/00-dev-concepts.md` — TypeScript 함수 오버로드, useSyncExternalStore, Tearing

### track — 자동 의존성 추적 HOC

컴포넌트를 감싸면 렌더 중 접근한 시그널을 자동으로 추적한다.

> 📍 packages/state-react/src/lib/track.ts

```tsx
export const EditLinkDialog = track(function EditLinkDialog({ onClose }) {
  const editor = useEditor()
  // editor.getXxx().get() 호출 시 자동으로 의존성 캡처
  // 해당 시그널이 변하면 이 컴포넌트만 리렌더
})
```

**구현**: JavaScript `Proxy`를 사용하여 컴포넌트 함수 호출을 가로채고, `useStateTracking`으로 의존성을 캡처한다. `React.memo`와 `React.forwardRef`도 올바르게 처리한다.

### useBreakpoint — 반응형 브레이크포인트

뷰포트 너비에 따라 UI를 적응시키는 시스템. `useValue`로 리액티브하게 동작한다.

> 📍 packages/tldraw/src/lib/ui/context/breakpoints.tsx
> 📍 packages/tldraw/src/lib/ui/constants.ts

```ts
export const PORTRAIT_BREAKPOINTS = [0, 389, 436, 476, 580, 640, 840, 1023]

export const PORTRAIT_BREAKPOINT = {
  ZERO: 0,          // 0px
  MOBILE_XXS: 1,    // 389px
  MOBILE_XS: 2,     // 436px
  MOBILE_SM: 3,     // 476px
  MOBILE: 4,        // 580px
  TABLET_SM: 5,     // 640px
  TABLET: 6,        // 840px
  DESKTOP: 7,       // 1023px
} as const
```

**BreakPointProvider** 내부에서 `editor.getViewportScreenBounds()`를 리액티브하게 읽어 현재 브레이크포인트를 계산한다:

```tsx
const breakpoint = useValue('breakpoint', () => {
  const { width } = editor?.getViewportScreenBounds() ?? { width: window.innerWidth }
  for (let i = 0; i < maxBreakpoint; i++) {
    if (width > PORTRAIT_BREAKPOINTS[i] && width <= PORTRAIT_BREAKPOINTS[i + 1]) {
      return i
    }
  }
  return maxBreakpoint
}, [editor])
```

**사용 예**: 브레이크포인트에 따라 UI 요소를 조건부로 표시/숨김한다:

```tsx
const breakpoint = useBreakpoint()

// 태블릿 이하에서만 모바일 스타일 패널 표시
{breakpoint < PORTRAIT_BREAKPOINT.TABLET_SM && !isReadonlyMode && (
  <MobileStylePanel />
)}

// 태블릿 이하에서 빠른 액션 표시 위치 변경
const showQuickActions = breakpoint < PORTRAIT_BREAKPOINT.TABLET
```

### CSS 네이밍 & z-index 레이어링

> 📍 packages/tldraw/src/lib/ui.css

**BEM 스타일 네이밍**: `tlui-{컴포넌트}__{부분}--{변형}`

```css
.tlui-button              /* 기본 버튼 */
.tlui-button__label       /* 버튼 라벨 */
.tlui-main-toolbar        /* 메인 툴바 */
.tlui-main-toolbar--horizontal  /* 수평 변형 */
.tlui-layout              /* 전체 UI 레이아웃 */
```

**z-index 레이어링 시스템**: CSS 변수로 계층을 관리한다:

```css
.tl-container {
  --tl-layer-above: 1;
  --tl-layer-focused-input: 10;
  --tl-layer-panels: 300;
  --tl-layer-menus: 400;
  --tl-layer-toasts: 650;
  --tl-layer-cursor: 700;
  --tl-layer-header-footer: 999;
  --tl-layer-following-indicator: 1000;
}
```

### CSS 미디어 쿼리 대신 JS 브레이크포인트를 쓰는 이유

CSS `@media` 쿼리는 **브라우저 창(window)** 크기만 기준으로 한다. tldraw 에디터가 사이드 패널 옆 600px 영역에 임베딩되어도, 브라우저 창이 1200px이면 "데스크톱"으로 판단한다. **에디터 컨테이너 크기**를 기준으로 해야 하므로 JS에서 직접 측정한다.

```
┌─ 브라우저 창 (1200px) ──────────────────────┐
│  ┌─ 사이드 패널 ─┐  ┌─ tldraw 에디터 ─────┐ │
│  │               │  │   (600px)            │ │
│  └───────────────┘  └──────────────────────┘ │
└──────────────────────────────────────────────┘
CSS @media: "1200px = 데스크톱" ← 틀림
JS 측정:    "600px = TABLET_SM" ← 정확
```

CSS에도 `@container` 쿼리가 있지만, JS 레벨에서 "컴포넌트를 아예 렌더링 안 함" 같은 제어가 필요하므로 JS 방식을 택했다.

### 컨테이너 크기 측정 → 시그널 연결 체인

DOM 컨테이너 크기 변화를 감지하여 시그널 스토어에 넣고, 브레이크포인트 → React 리렌더까지 연결하는 전체 파이프라인.

> 📍 packages/editor/src/lib/hooks/useScreenBounds.ts
> 📍 packages/editor/src/lib/editor/Editor.ts:3784-3851

**Step 1: ResizeObserver로 컨테이너 감시** — `DefaultCanvas`에서 캔버스 div ref를 넘겨 호출한다.

```ts
// useScreenBounds.ts
useLayoutEffect(() => {
  const updateBounds = throttle(() => {
    editor.updateViewportScreenBounds(ref.current)
  }, 200)

  // 3중 감시 (edge case 대응)
  editor.timers.setInterval(updateBounds, 1000)           // 1초 폴링
  win.addEventListener('resize', updateBounds)              // window resize
  const resizeObserver = new ResizeObserver(() => updateBounds())
  resizeObserver.observe(container)                         // ResizeObserver
}, [editor, ref])
```

**Step 2: DOM 측정 → 스토어 저장**

```ts
// Editor.ts:3784
updateViewportScreenBounds(screenBounds: Box | HTMLElement) {
  const rect = screenBounds.getBoundingClientRect()  // DOM 측정
  this.updateInstanceState({ screenBounds: screenBounds.toJson() })
  //   ↑ store.put() → TLInstance 레코드 갱신 → 시그널 체인 시작
}
```

**Step 3: @computed 시그널 전파**

```ts
// Editor.ts:3848
@computed getViewportScreenBounds() {
  const { x, y, w, h } = this.getInstanceState().screenBounds
  return new Box(x, y, w, h)
}
```

**Step 4: useValue가 React 리렌더로 변환** — BreakPointProvider 내부에서 `getViewportScreenBounds()`를 구독.

```
ResizeObserver → getBoundingClientRect() → store.put()
  → @computed 시그널 갱신 → EffectScheduler 변경 감지
    → useSyncExternalStore → React 리렌더
```

컨테이너 리사이즈는 드문 이벤트(창 리사이즈, 패널 토글, 기기 회전)라서 throttle 200ms와 한 프레임 지연이 문제되지 않는다. 캔버스 자체는 CSS `width: 100%`로 즉시 리사이즈되고, 한 프레임 늦는 건 툴바/미니맵 같은 UI 오버레이뿐이다.

### 시그널-to-UI 전체 흐름 요약

```
1. Editor 내부 시그널 변경
   editor.setCurrentPage(newPage)
       ↓
2. Atom/Computed 값 갱신, 글로벌 epoch 증가
       ↓
3. EffectScheduler가 의존성 변경 감지
   haveParentsChanged() → true
       ↓
4. scheduleEffect() → useSyncExternalStore의 notify() 호출
       ↓
5. React가 스냅샷 비교 → 변경된 컴포넌트만 리렌더
```

> 📍 관련: `.study/06-state-management.md` — 시그널 시스템, 의존성 자동 추적, epoch 기반 변경 감지
