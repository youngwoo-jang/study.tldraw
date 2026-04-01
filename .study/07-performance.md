# 성능 최적화

## useShallowObjectIdentity — SDK 방어적 참조 안정화

SDK 소비자가 인라인 객체를 prop으로 넘길 때 발생하는 불필요한 리렌더를 라이브러리 내부에서 흡수하는 패턴.

> 📍 packages/editor/src/lib/hooks/useIdentity.tsx

```tsx
function useShallowObjectIdentity<T extends object>(obj: T): T {
    const ref = useRef(obj)
    if (areNullableObjectsShallowEqual(obj, ref.current)) {
        return ref.current  // 내용 같으면 이전 참조 반환
    }
    ref.current = obj
    return obj
}
```

### 문제 상황

```tsx
function App() {
  const [count, setCount] = useState(0)
  return <Tldraw components={{ Toolbar: MyToolbar }} />
  // count 변경 → App 리렌더 → { Toolbar: MyToolbar } 새 객체 생성
  // 내용은 같지만 참조가 다름 (0x001 !== 0x002)
}
```

이 새 참조가 `useMemo`의 의존성으로 들어가면 context value가 매번 재생성되고, 모든 consumer가 리렌더된다.

### 해결

React 기본 비교(`Object.is`)는 참조만 보지만, `useShallowObjectIdentity`는 한 단계 더 들어가서 프로퍼티별로 비교한다. 내용이 같으면 이전 참조를 반환하여 downstream의 `useMemo`가 재계산되지 않게 한다.

```
React 기본:  { Toolbar: MyToolbar } !== { Toolbar: MyToolbar }  → 변경됨
shallow 비교: .Toolbar === .Toolbar                              → 이전 참조 유지
```

### 왜 직접 구현했나

`use-deep-compare-effect` 같은 라이브러리가 있지만:
- deep이 아니라 shallow만 필요 (컴포넌트 참조는 1단계 비교로 충분)
- 구현이 10줄 미만 — 외부 의존성 추가할 정도가 아님
- SDK라서 의존성 최소화가 중요 (번들 사이즈, 공급망 리스크)

> 📍 관련: `.study/02-ui-architecture.md` — 컴포넌트 오버라이드 시스템에서의 사용 맥락

---

## 자체 시그널 시스템을 만든 이유 — Zustand 등 범용 라이브러리 대비

캔버스 에디터는 일반 웹앱과 요구사항이 다르다. 수천 개의 도형, 60fps 인터랙션, React 바깥 로직이 많아서 범용 상태관리 라이브러리로는 성능이 안 나온다.

### 1. 시그널의 세밀한 반응성 vs store 단위 구독

```ts
// Zustand: store가 바뀔 때마다 selector 실행 → 비교
const x = useStore(s => s.shapes[id].x)

// tldraw 시그널: shape.x에 직접 의존 → 그것만 바뀔 때 반응
const x = useValue('x', () => editor.getShape(id).x, [id])
```

Zustand는 store가 바뀌면 관련 selector를 전부 실행해서 비교한다. 시그널은 의존성 그래프가 정확히 누가 누구에 의존하는지 알고 있으므로 관련 없는 컴포넌트는 selector 실행조차 안 한다.

### 2. React 바깥에서도 독립 동작

Editor 내부 로직(도형 충돌 계산, 바인딩 업데이트 등)은 React 없이 순수 시그널 그래프로 돌아간다. Zustand는 근본적으로 React hook 기반이라 이런 용도에 안 맞는다.

### 3. Epoch 기반 O(1) 변경 감지

Zustand는 값 비교(`===` 또는 shallow equal)로 변경을 감지한다. 시그널은 정수 비교 한 번으로 끝난다. 60fps로 도형을 드래그할 때 이 차이가 크다.

> 📍 관련: `.study/06-state-management.md` — epoch 기반 변경 감지

---

## Epoch 기반 캐싱 — Computed 체인 최적화

Computed의 getter를 매번 실행하지 않고, **부모 epoch 비교로 실행 여부를 먼저 판단**하는 것이 핵심 최적화 포인트.

> 📍 packages/state/src/lib/Computed.ts:272-290

```
display.get() 호출
  → 부모들의 epoch 비교 (정수 비교 몇 번)
    → 전부 같음 → 캐시 반환. getter 실행 안 함.
    → 하나라도 다름 → getter 실행 + 의존성 재수집
```

### Computed 체인에서의 효과

```ts
const visibleShapes = computed('visible', () => {
    return allShapes.get().filter(s => viewport.get().contains(s.bounds))
})
const sortedShapes = computed('sorted', () => {
    return visibleShapes.get().sort((a, b) => a.z - b.z)
})
const rendered = computed('rendered', () => {
    return sortedShapes.get().map(s => computePath(s))
})
```

viewport가 살짝 움직여도 보이는 도형 목록이 같은 경우:

```
viewport 변경
  → visibleShapes.get() → epoch 다름 → getter 실행 (어쩔 수 없음)
  → 결과가 같은 도형 목록 → visibleShapes.lastChangedEpoch 안 바뀜
  → sortedShapes.get() → epoch 비교 → 안 바뀜 → 캐시 반환 ← 절약!
  → rendered.get()     → epoch 비교 → 안 바뀜 → 캐시 반환 ← 절약!
```

체인이 길고 각 단계가 무거울수록 epoch 캐싱의 효과가 커진다. 도형 수천 개가 있어도 안 바뀐 것들은 정수 비교만 하고 끝난다.

> 📍 관련: `.study/06-state-management.md` — Computed lazy evaluation, 의존성 자동 추적

---

## useQuickReactor — React 리렌더 우회 DOM 직접 수정

도형의 CSS transform, display, opacity 등 빈번히 바뀌는 속성을 React 상태가 아닌 **DOM API 직접 호출**로 업데이트하는 패턴.

> 📍 packages/state-react/src/lib/useQuickReactor.ts
> 📍 packages/editor/src/lib/components/Shape.tsx:63-105

```ts
// React 방식: 데이터 → setState → render() → VDOM diff → DOM
// Quick 방식: 데이터 → 시그널 감지 → element.style.setProperty() 직접 호출
useQuickReactor('set shape stuff', () => {
  const transform = Mat.toCssString(editor.getShapePageTransform(id))
  if (transform !== prev.transform) {
    setStyleProperty(container, 'transform', transform)  // DOM 직접 수정
    prev.transform = transform
  }
})
```

`useQuickReactor`는 시그널 변경 시 **즉시 동기 실행**. 반면 `useReactor`는 `throttleToNextFrame`으로 다음 rAF까지 지연(batching 가능). transform처럼 한 프레임 지연이 눈에 보이는 속성은 Quick이어야 한다.

> 📍 관련: `.study/06-state-management.md` — 시그널 시스템

---

## CSS Containment — 리플로우 체인 차단

`contain` 속성으로 DOM 리플로우(레이아웃 재계산)의 전파 경계를 끊는다. 3단계로 겹쳐 적용:

> 📍 packages/editor/editor.css

```
.tl-canvas (contain: strict)            ← 캔버스 밖으로 절대 안 나감
  └── .tl-html-layer (contain: layout style size)  ← 레이어 간 전파 차단
        └── .tl-shape (contain: size layout)        ← 도형 간 전파 차단
```

도형 하나의 크기가 변해도 다른 도형의 레이아웃을 재계산하지 않는다. 안쪽 벽이 뚫려도 바깥 벽이 막아주는 구조.

브라우저는 이런 최적화를 자동으로 할 수 없다. DOM만 보고 "자식이 부모 크기에 영향을 주는지" 판단 불가. 개발자가 `contain`으로 명시적으로 보장해야 한다.

> 📍 상세: `.study/10-canvas-rendering.md` — CSS Containment로 리플로우 격리

---

## GPU 가속 — 확정 트리거 회피, 힌트만 제공

`will-change: transform`, `translateZ(0)` 같은 확정 GPU 레이어 승격 트리거를 의도적으로 안 쓴다.

GPU 레이어는 공짜가 아니다:
- 레이어당 텍스처 메모리(VRAM) 할당 (500×300 요소 = ~600KB)
- 매 프레임 모든 레이어를 합성(composite)하는 비용
- 도형 1000개에 각각 `will-change: transform` → ~600MB VRAM

대신 `.tl-canvas`에만 `contain: strict` + `content-visibility: auto`를 적용하여 브라우저 휴리스틱에 맡긴다. GPU를 적극 활용하는 전략이 아니라 CPU 쪽에서 할 일을 최소화(culling, contain, React 우회)해서 GPU 가속 없이도 충분하게 만든 구조.

> 📍 상세: `.study/10-canvas-rendering.md` — GPU 가속 전략

---

## R-tree 공간 인덱싱 — Culling 성능의 핵심

**RBush** 라이브러리(R-tree JS 구현)로 모든 도형의 바운딩 박스를 공간 인덱싱. 뷰포트 영역 쿼리로 O(log n)에 보이는 도형을 찾아 나머지를 `display: none`으로 숨긴다.

> 📍 packages/editor/src/lib/editor/managers/SpatialIndexManager/SpatialIndexManager.ts
> 📍 packages/editor/src/lib/editor/derivations/notVisibleShapes.ts
> 📍 packages/editor/src/lib/hooks/useShapeCulling.tsx

R-tree는 가까운 도형끼리 계층적 사각형으로 묶은 트리. 쿼리 시 뷰포트와 안 겹치는 그룹은 통째로 스킵하여 개별 비교를 줄인다. 벌크 로딩 시 STR(Sort-Tile-Recursive) 알고리즘으로 X축 정렬 → Y축 정렬 → 타일 분할.

바운딩 박스는 DOM이 아닌 **tldraw 데이터에서 수학적으로 계산**. 텍스트만 예외적으로 숨겨진 측정용 div에서 `getBoundingClientRect()` 호출 후 캐싱.

단일 `CullingController` reactor가 모든 도형의 display를 한번에 갱신 (O(1) 구독, 도형별 구독 아님).

> 📍 상세: `.study/10-canvas-rendering.md` — Culling, R-tree
