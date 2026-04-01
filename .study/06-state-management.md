# 상태 관리

## 시그널 (Signal) — 변경을 자동 전파하는 반응형 값 컨테이너

tldraw의 상태 관리 핵심. `@tldraw/state` 패키지에 구현되어 있다. 두 가지 종류가 있다.

> 📍 packages/state/src/lib/types.ts:72 — Signal 인터페이스
> 📍 packages/state/src/lib/Atom.ts — 원시 시그널
> 📍 packages/state/src/lib/Computed.ts — 파생 시그널

### Atom — 직접 읽고 쓸 수 있는 원시 시그널

```ts
const count = atom('count', 0)

count.get()              // 0 — 읽기 (의존성 자동 캡처)
count.set(5)             // 쓰기 → globalEpoch 증가 → 자식들에게 통지
count.update(n => n + 1) // 함수형 업데이트
```

`get()` 내부에서 `maybeCaptureParent(this)`를 호출하여 "누가 나를 읽었는지" 기록한다. `set()` 시에는 동일 값이면 무시(no-op)하고, 다르면 `advanceGlobalEpoch()` → `atomDidChange()` → 의존 그래프 순회 → 리액터 실행.

### Computed — 다른 시그널로부터 파생되는 게으른(lazy) 시그널

```ts
const firstName = atom('first', 'John')
const lastName = atom('last', 'Doe')

const fullName = computed('fullName', () => {
    return firstName.get() + ' ' + lastName.get()
    //     ^^^ .get() 호출 시 자동으로 부모 의존성 캡처
})
```

**핵심 특성: lazy evaluation** — `computed()`를 만들 때가 아니라 누군가 `.get()`으로 값을 읽을 때 비로소 getter(= derive 함수)를 실행한다. 이후에는 부모 epoch이 바뀌지 않았으면 캐시된 값을 반환하고, 바뀌었을 때만 getter를 다시 실행한다.

> 📍 관련: `.study/07-performance.md` — epoch 기반 캐싱 최적화

---

## Epoch 기반 변경 감지

모든 시그널의 변경 여부를 정수 비교 한 번으로 판단하는 시스템.

> 📍 packages/state/src/lib/transactions.ts:74-83 — globalEpoch
> 📍 packages/state/src/lib/Atom.ts:106 — lastChangedEpoch

### 구조

```
globalEpoch: 전역 카운터. atom.set() 할 때마다 +1
lastChangedEpoch: 각 시그널이 마지막으로 변경된 시점의 epoch

name.set('Bob')     → globalEpoch = 1, name.lastChangedEpoch = 1
age.set(30)         → globalEpoch = 2, age.lastChangedEpoch = 2
name.set('Charlie') → globalEpoch = 3, name.lastChangedEpoch = 3
```

### Computed에서의 활용

Computed는 부모를 등록할 때 그 시점의 epoch도 함께 기록한다.

```
display의 부모: [showDetails, name]
기록된 epoch:   [1,           3]    ← "내가 마지막으로 봤을 때의 값"
```

나중에 `display.get()` 호출 시:

```ts
// helpers.ts — haveParentsChanged()
for (let i = 0; i < child.parents.length; i++) {
    if (child.parents[i].lastChangedEpoch !== child.parentEpochs[i]) {
        return true  // 숫자가 다르면 바뀐 것
    }
}
return false
```

**값 자체를 비교하지 않고 정수 하나만 비교**하므로, 도형 수천 개가 있어도 안 바뀐 것들은 정수 비교만 하고 끝난다.

---

## 의존성 자동 추적 시스템

"실행해보면 뭘 읽었는지 알 수 있다" — getter 함수를 실행하면서 그 안에서 `.get()`이 호출된 시그널을 자동 수집하는 메커니즘.

> 📍 packages/state/src/lib/capture.ts

### 왜 필요한가

```ts
const view = computed('view', () => {
    if (isSelected.get()) {
        return `${name.get()} (${age.get()})`
    }
    return 'not selected'
})
```

개발자가 의존성을 선언하지 않았다. getter 함수만 넘겼을 뿐. 시스템이 "view는 isSelected, name, age에 의존한다"는 걸 어떻게 아는가? → **getter를 실행해보면 안다.**

### 핵심 트릭: `.get()` 안에 추적 코드를 심어둠

```ts
// Atom.ts:143
get() {
    maybeCaptureParent(this)  // ← 이 한 줄이 전부
    return this.current
}
```

모든 시그널의 `.get()`에 `maybeCaptureParent`가 들어있다. 전역 캡처 스택이 활성화되어 있으면 "나 읽혔어" 하고 등록하고, 비활성이면 아무것도 안 한다.

### 전역 캡처 스택 = 추적 모드 스위치

```ts
// capture.ts:16
const inst = singleton('capture', () => ({ stack: null as null | CaptureStackFrame }))
```

```
stack = null  → 추적 모드 OFF. .get()해도 기록 안 함
stack = Frame → 추적 모드 ON. .get()하면 부모로 등록
```

### 3단계 흐름

```
① startCapturingParents(display)   → 스택에 프레임 push (추적 ON)
    parentSet.clear()               → 기존 의존성 초기화

② getter 함수 실행
    → showDetails.get() → maybeCaptureParent → parents[0] = showDetails
    → name.get()        → maybeCaptureParent → parents[1] = name

③ stopCapturingParents()            → 스택에서 pop (추적 OFF)
    → 이번에 안 읽힌 부모는 detach (연결 해제)
```

### 매번 새로 수집 → 조건부 의존성 자동 처리

getter를 실행할 때마다 의존성을 새로 수집한다. 최초 1회만이 아니다.

```
1회차: showDetails=true
  → getter 실행 → 의존성: [showDetails, name, age]

--- showDetails.set(false) → display 재계산 ---

2회차: showDetails=false
  → getter 실행 → showDetails.get()만 호출됨, name/age는 안 읽힘
  → 의존성: [showDetails] ← name, age 자동 제거!
```

이제 `name.set('Bob')`을 해도 display는 재계산하지 않는다. 수동 선언으로는 이런 동적 의존성을 처리하기 거의 불가능하다.

### 스택 중첩 — Computed 안에서 다른 Computed를 읽는 경우

```ts
const a = atom('a', 1)
const b = computed('b', () => a.get() * 2)
const c = computed('c', () => b.get() + 10)
```

```
c.get():
  stack: Frame(c)
    b.get():
      stack: Frame(b) → Frame(c)    ← push
        a.get() → a가 b의 부모로 등록
      stopCapturing → stack: Frame(c)  ← pop
      b가 c의 부모로 등록
  stopCapturing → stack: null

결과: b.parents = [a], c.parents = [b]
```

`a`가 변해도 `b`의 계산 결과가 같으면 `c`는 재계산하지 않는다.

---

## `__unsafe__getWithoutCapture()` — 추적 없이 값만 읽기

```ts
// .get() = 값 읽기 + 의존성 추적
get() {
    maybeCaptureParent(this)
    return this.current
}

// .__unsafe__getWithoutCapture() = 값만 읽기
__unsafe__getWithoutCapture() {
    return this.current  // 추적 없음
}
```

이미 다른 경로로 시그널 변경을 감지하고 있을 때 사용한다. 예를 들어 `useValue`에서는 `subscribe`에서 `react()`로 변경을 감지하고 있으므로, 최종 값 반환 시 불필요한 추적을 건너뛴다.

> 📍 packages/state-react/src/lib/useValue.ts:107

---

## track vs useValue — 시그널-React 연결의 두 가지 방식

tldraw 시그널은 React를 모른다. 시그널이 바뀌어도 React 컴포넌트는 리렌더되지 않는다. 이 간극을 메우는 두 가지 방법이 있다.

> 📍 packages/state-react/src/lib/track.ts
> 📍 packages/state-react/src/lib/useValue.ts
> 📍 packages/state-react/src/lib/useStateTracking.ts

### 왜 필요한가 — React는 setState로만 리렌더된다

```ts
const color = atom('color', 'red')
color.set('blue')  // ← React는 이걸 모른다. 리렌더 안 됨
```

누군가 시그널 변경을 감시하다가 React에게 "다시 그려"라고 알려야 한다. 이 역할을 `track`과 `useValue`가 한다.

### `useValue` — 수동으로 개별 시그널 구독

```tsx
function MyComponent() {
  const color = useValue('color', () => editor.user.getColor(), [editor])
  const name = useValue('name', () => editor.user.getName(), [editor])
  const zoom = useValue('zoom', () => editor.getZoomLevel(), [editor])
  return <div>{color} {name} {zoom}</div>
}
```

시그널마다 하나씩 `useValue`를 호출해야 한다. 각각이 독립된 `useSyncExternalStore` 구독을 건다.

### `track` — 컴포넌트 전체를 감싸서 자동 구독

```tsx
const MyComponent = track(function MyComponent() {
  // 그냥 읽기만 하면 전부 자동 구독
  const color = editor.user.getColor()
  const name = editor.user.getName()
  const zoom = editor.getZoomLevel()
  return <div>{color} {name} {zoom}</div>
})
```

렌더 함수 전체를 `EffectScheduler`로 감싸서, 실행 중 `.get()`이 호출된 시그널을 전부 자동 수집하고, 그 중 하나라도 바뀌면 React 리렌더를 건다.

### 비교

| 기준 | `track` | `useValue` |
|------|---------|-----------|
| 적용 단위 | 컴포넌트 전체 | 개별 시그널 |
| 추적 방식 | 렌더 중 `.get()` 호출 자동 캡처 | 명시적으로 지정한 시그널만 |
| 사용 위치 | 컴포넌트 바깥에서 래핑 | 컴포넌트 안에서 hook 호출 |
| 구독 수 | 컴포넌트당 1개 스케줄러 | hook 호출마다 개별 구독 |

**사용 기준**: 시그널을 여러 개 쓰면 `track`이 편하고, 하나만 쓰거나 부분적으로 구독하려면 `useValue`가 적합하다. `track`을 쓰면 `useValue`는 필요 없다.

### 주의: 시그널 쓰는 컴포넌트마다 개별로 `track`을 감싸야 한다

Context Provider처럼 root에서 한 번 감싸는 게 아니다. 시그널을 읽는 각 컴포넌트에 직접 적용해야 한다.

```tsx
// 각 컴포넌트가 개별적으로 track으로 감싸져 있다
export const DefaultToolbar = track(function DefaultToolbar() { ... })
export const DefaultNavigationPanel = track(function DefaultNavigationPanel() { ... })
export const DefaultMinimap = track(function DefaultMinimap() { ... })
```

안 감싸면 그 컴포넌트는 시그널이 바뀌어도 리렌더 안 된다.

### Zustand와의 차이 — selector 없이 자동 추적이 가능한 이유

Zustand는 상태가 **평범한 JS 객체**라서 `state.color`를 읽어도 누가 읽었는지 알 방법이 없다. 그래서 selector가 필수다:

```ts
const color = useStore(store, (s) => s.color)  // selector로 명시
```

tldraw 시그널은 반드시 **`.get()` 메서드**를 통해 읽어야 하고, 그 안에 `maybeCaptureParent(this)` 트랩이 있다. `track`이 렌더 전체를 캡처 컨텍스트로 감싸놨기 때문에 `.get()` 호출만으로 자동 등록된다. 대신 `editor.color` 같은 직접 접근은 안 되고, 항상 `editor.getColor()` 같은 메서드를 거쳐야 한다.

---

## track HOC — 구현 원리 상세

Proxy와 EffectScheduler를 조합하여 컴포넌트 렌더 중 시그널 접근을 자동 추적한다.

> 📍 packages/state-react/src/lib/track.ts:109-123
> 📍 packages/state-react/src/lib/useStateTracking.ts:29-85

### Step 1: Proxy로 함수 호출 가로채기

React 함수형 컴포넌트는 그냥 함수다. React가 `MyComponent(props)`를 호출하면 Proxy의 `apply` 트랩이 가로챈다:

```ts
// track.ts:21-38
export const ProxyHandlers = {
  apply(Component, thisArg, argumentsList) {
    return useStateTracking(Component.name, () =>
      Component.apply(thisArg, argumentsList)
    )
  }
}

// track.ts:109-123
export function track(baseComponent) {
  return memo(new Proxy(baseComponent, ProxyHandlers))
  //     ^^^^ React.memo로도 감싸서 props 변경 없으면 리렌더 방지
}
```

Proxy는 시그널 접근을 가로채는 게 아니라, **컴포넌트 함수 호출 자체를 가로채서** 앞뒤에 `useStateTracking` 로직을 끼우는 데 쓰인다.

### Step 2: useStateTracking이 EffectScheduler를 세팅

```ts
// useStateTracking.ts (simplified)
function useStateTracking(name, render) {
  const [scheduler, subscribe, getSnapshot] = useMemo(() => {
    let scheduleUpdate = null

    const scheduler = new EffectScheduler(name, () => renderRef.current(), {
      scheduleEffect() {
        scheduleUpdate?.()     // 시그널 바뀌면 React한테 알림
      }
    })

    return [
      scheduler,
      (cb) => { scheduleUpdate = cb; return () => { scheduleUpdate = null } },
      () => scheduler.scheduleCount
    ]
  }, [name])

  useSyncExternalStore(subscribe, getSnapshot, getSnapshot)

  useEffect(() => {
    scheduler.attach()           // 시그널 변경 감시 시작
    scheduler.maybeScheduleEffect()
    return () => scheduler.detach()
  }, [scheduler])

  return scheduler.execute()     // 렌더 실행 + 의존성 캡처
}
```

### Step 3: scheduler.execute()가 의존성을 캡처

```ts
// EffectScheduler.ts:157-170
execute() {
  startCapturingParents(this)   // 전역 캡처 스택에 등록
  const result = this.runEffect()  // 컴포넌트 렌더 함수 실행
  stopCapturingParents()         // 캡처 종료
  return result
}
```

렌더 함수 실행 중 `atom.get()` → `maybeCaptureParent()` → 스케줄러의 `parents[]`에 등록. 렌더가 끝나면 `stopCapturingParents()`에서 이번에 안 읽힌 시그널은 자동 `detach`.

### Step 4: 리렌더 유도 — useSyncExternalStore

`useSyncExternalStore`는 React 18 내장 API로, 외부 상태 변경을 React 리렌더로 연결한다:

1. React가 `subscribe(callback)`을 호출 → 콜백을 `scheduleUpdate`에 저장
2. 시그널 변경 → `scheduler.scheduleEffect()` → `scheduleCount++` → `scheduleUpdate()` 호출
3. React가 `getSnapshot()` 호출 → `scheduleCount`가 바뀜 → 리렌더

원리는 `setState(n => n+1)` dummy state와 같다. 숫자를 올려서 리렌더를 유도하되, `useSyncExternalStore`가 concurrent mode에서 안전하게 동기화해준다.

### 추적 단위 = 컴포넌트 인스턴스

`useMemo` 안에서 `EffectScheduler`가 생기므로 **컴포넌트 인스턴스당 1개** 스케줄러가 존재한다:

```tsx
<A />   // scheduler 인스턴스 1 → color 구독
<A />   // scheduler 인스턴스 2 → color 구독 (독립)
<B />   // scheduler 인스턴스 3 → zoom 구독
// color 변경 → A 두 개만 리렌더, B는 무관
```

### track은 Computed와 다르다 — 캐싱이 없다

Computed는 부모 안 바뀌면 재계산 안 하고 캐시된 값을 반환한다. `track` 컴포넌트는 리렌더가 트리거되면 **항상 렌더 함수를 다시 실행**한다. "안 바뀌면 리렌더 안 한다"는 것이지, "리렌더하되 캐시를 쓴다"는 게 아니다. 캐싱 역할은 `React.memo`가 props 비교로 담당한다.

```
시그널 변경
  → maybeScheduleEffect()
    → epoch 비교: 부모 안 바뀜 → 리렌더 안 함
    → epoch 비교: 부모 바뀜 → React 리렌더
      → scheduler.execute() → 렌더 함수 풀 실행 + 의존성 재캡처
```

---

## useValue — 시그널을 React에 연결하는 hook

시그널 시스템과 React 사이의 브릿지. `useSyncExternalStore` 기반.

> 📍 packages/state-react/src/lib/useValue.ts
> 📍 상세: `.study/02-ui-architecture.md` — 반응형 UI 섹션

### TypeScript 함수 오버로드로 두 가지 사용법 제공

```ts
// 오버로드 1: 시그널 직접 구독
export function useValue<Value>(value: Signal<Value>): Value

// 오버로드 2: computed 생성 + 구독
export function useValue<Value>(name: string, fn: () => Value, deps: unknown[]): Value

// 구현부 (런타임에 이것만 존재)
export function useValue() {
    // args.length로 어떤 오버로드인지 분기
}
```

### 내부 흐름

```
시그널 값 변경
  → react() 콜백에서 $val.get()으로 변경 감지
  → notify() 호출 (= useSyncExternalStore의 subscribe 콜백)
  → React가 getSnapshot() 호출 → lastChangedEpoch 비교 → 다르면 리렌더
  → $val.__unsafe__getWithoutCapture()로 값 반환 (추적 없이)
```

### `useSyncExternalStore`의 역할

React 바깥 상태를 React state처럼 취급하게 해주는 React 18 hook. 이걸 통해 연결된 외부 값은 concurrent mode에서 tearing 없이 안전하게 읽힌다.

> 📍 관련: `.study/00-dev-concepts.md` — React Concurrent Mode, Tearing, useSyncExternalStore

---

## @computed 데코레이터 — 메서드를 Computed 시그널로 변환

TypeScript 데코레이터를 사용하여 클래스 메서드를 **캐시되는 반응형 Computed 시그널**로 만든다.

> 📍 packages/state/src/lib/computed.ts

```ts
// 일반 메서드 — 호출할 때마다 매번 새로 계산
getViewportScreenBounds() {
  const { x, y, w, h } = this.getInstanceState().screenBounds
  return new Box(x, y, w, h)
}

// @computed — 의존하는 값이 안 바뀌면 캐시된 결과 반환
@computed getViewportScreenBounds() {
  const { x, y, w, h } = this.getInstanceState().screenBounds
  return new Box(x, y, w, h)
}
```

두 가지를 해준다:

1. **메모이제이션** — 내부에서 읽은 시그널이 안 바뀌었으면 이전 결과를 그대로 반환
2. **의존성 추적** — 내부에서 읽은 시그널이 바뀌면 이걸 구독하는 쪽에 변경 전파

내부적으로 메서드를 getter로 교체하되, 인스턴스별로 `Computed` 시그널 객체를 생성하여 캐싱과 의존성 추적을 자동으로 붙인다. `computed()` 함수를 데코레이터 형태로 쓸 수 있게 한 것.

> 📍 관련: `.study/00-dev-concepts.md` — TypeScript 데코레이터 문법, 데코레이터 vs Proxy 비교
