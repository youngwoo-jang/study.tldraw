# 캔버스 & 렌더링

## DOM 기반 렌더링 — Canvas API를 쓰지 않는 무한 캔버스

tldraw는 HTML `<canvas>` 요소가 아닌 **DOM(div + svg)**으로 도형을 렌더링한다. 각 도형이 React 컴포넌트이며, ShapeUtil의 `component()` 메서드가 JSX를 반환한다.

> 📍 packages/editor/src/lib/components/default-components/DefaultCanvas.tsx
> 📍 packages/editor/src/lib/editor/shapes/ShapeUtil.ts:193

### 캔버스 DOM 구조

```
<div class="tl-canvas">                              ← div (canvas 아님)
  <svg class="tl-svg-context">                        ← SVG defs만 (패턴, 그라디언트)
  <div class="tl-html-layer tl-shapes">               ← 카메라 transform 컨테이너
    <div class="tl-shape" style="transform: matrix(...)">  ← 도형 하나 = div 하나
      <SVGContainer> 또는 <HTMLContainer>                   ← 도형 내용물
    </div>
  </div>
  <div class="tl-overlays">                           ← 선택 표시, 핸들
```

### 도형별 렌더링 방식

ShapeUtil의 `component()`가 React JSX를 반환한다:

```tsx
// 기하 도형 → SVG + HTML
class GeoShapeUtil {
  component(shape) {
    return (
      <>
        <SVGContainer>{/* <path>, <rect> 등 */}</SVGContainer>
        <HTMLContainer>{/* 텍스트 라벨 */}</HTMLContainer>
      </>
    )
  }
}

// 비디오 → 그냥 <video> 태그
class VideoShapeUtil {
  component(shape) {
    return <HTMLContainer><video src={...} /></HTMLContainer>
  }
}
```

### 자유 그리기도 SVG path

연필(Draw) 도구의 자유 곡선도 `<canvas>` 드로잉이 아니라 SVG `<path>` 요소다.

> 📍 packages/tldraw/src/lib/shapes/draw/DrawShapeUtil.tsx
> 📍 packages/tldraw/src/lib/shapes/shared/freehand/svgInk.ts

입력 포인트 배열 → `getStrokePoints()` → `setStrokePointRadii()`(압력 반영) → `svgInk()`로 좌/우 윤곽 트랙을 가진 닫힌 path 생성:

```
     좌측 윤곽 트랙
    ╭───────────────╮
캡 ○                ○ 캡
    ╰───────────────╯
     우측 윤곽 트랙
```

결과물은 `<path d="M ... Q ... T ... Z" fill="black" />`.

### CSS `matrix(a, b, c, d, e, f)` — 도형 배치의 핵심

각 도형의 이동+회전+스케일을 **6개 숫자의 행렬 하나**로 표현한다.

> 📍 packages/editor/src/lib/primitives/Mat.ts:269-273
> 📍 packages/editor/src/lib/components/Shape.tsx:63-105

```
┌ a  c  e ┐     a = scaleX × cos(회전)    e = translateX
│ b  d  f │     b = scaleX × sin(회전)    f = translateY
└ 0  0  1 ┘     c = -scaleY × sin(회전)
                 d = scaleY × cos(회전)
```

예: `x=100, y=50, 회전=0` → `matrix(1, 0, 0, 1, 100, 50)`

**translate/rotate/scale 따로 안 쓰는 이유**: 부모-자식 변환 합성 때문. 행렬끼리는 곱셈 한 번으로 합성 가능:

```ts
// 프레임 안의 도형 → 행렬 곱셈으로 최종 위치 산출
pageTransform = Mat.Compose(parentPageTransform, childLocalTransform)
```

`getShapeLocalTransform()`은 도형 자신의 x, y, rotation으로 로컬 행렬을, `getShapePageTransform()`은 부모 체인을 재귀적으로 합성한 페이지 행렬을 반환한다. 페이지 행렬은 `ComputedCache`로 캐싱.

**데이터 → CSS 파이프라인**:

```
shape { x, y, rotation }
  → Mat.Identity().translate(x, y).rotate(rotation)     // 로컬 행렬
  → Mat.Compose(parentPageTransform, localTransform)     // 페이지 행렬 (캐싱)
  → Mat.toCssString()  // "matrix(a, b, c, d, e, f)"    // toDomPrecision: 소수점 4자리
  → setStyleProperty(element, 'transform', ...)          // DOM 직접 수정 (React 우회)
```

이 과정은 `useQuickReactor`로 실행되어 React 리렌더 사이클을 거치지 않는다. 이전 transform 문자열과 비교해서 바뀐 경우에만 DOM에 반영.

### 카메라/줌 = 부모 레이어의 CSS transform

Canvas API의 뷰포트 수학이 아니라, **레이어 div 하나에** CSS `scale()` + `translate()` 적용:

> 📍 packages/editor/src/lib/components/default-components/DefaultCanvas.tsx:57-96

```ts
const { x, y, z } = editor.getCamera()  // z = 줌 레벨
const offset = z >= 1
  ? modulate(z, [1, 8], [0.125, 0.5], true)
  : modulate(z, [0.1, 1], [-2, 0.125], true)

const transform = `scale(${z}) translate(${x + offset}px, ${y + offset}px)`
setStyleProperty(rHtmlLayer.current, 'transform', transform)
```

**핵심**: 패닝/줌 시 **개별 도형의 CSS를 전혀 건드리지 않는다**. 부모 레이어의 transform만 바꾸면 CSS가 자식 전체에 자동 적용:

```
도형 1000개의 transform 각각 업데이트 → DOM 접근 1000번 ❌
부모 레이어 transform 하나만 업데이트 → DOM 접근 1번 ✅
```

**transform 순서**: `scale(z) translate(x, y)` — scale 안에서 translate가 적용되므로 translate 값이 페이지 좌표계 기준이 되어 계산이 단순해진다.

**offset**: `.tl-html-layer`가 `width: 1px`이라 줌 레벨에 따라 서브픽셀 반올림 오차 발생. HTML 레이어와 SVG 오버레이 간 정렬 보정값.

### 유일한 Canvas 2D API 사용처 — CanvasShapeIndicators

도형 선택 시 나타나는 **파란 윤곽선만** Canvas 2D API로 그린다. 도형 자체가 아닌 UI 피드백용.

> 📍 packages/editor/src/lib/components/default-components/CanvasShapeIndicators.tsx

`<canvas>` 1개에 모든 선택 인디케이터를 일괄 렌더링:

```ts
const ctx = canvas.getContext('2d')
// 협업자 인디케이터 (0.7 opacity)
// 선택/호버 인디케이터 (1.5px stroke)
// 힌트 인디케이터 (2.5px stroke)
```

각 ShapeUtil은 `getIndicatorPath()`로 `Path2D` 객체를 반환한다:

```ts
override getIndicatorPath(shape) {
  const path = new Path2D()
  path.rect(0, 0, bounds.width, bounds.height)
  return path
}
```

**왜 인디케이터만 Canvas인가**: 다중 선택 시 SVG 요소 100개 생성+리플로우 vs Canvas에 `ctx.stroke()` 100번. 인디케이터는 클릭/입력이 필요 없는 순수 시각 피드백이라 DOM의 장점이 불필요하고 Canvas의 일괄 렌더링이 유리하다.

**기즈모(리사이즈/회전 핸들)는 DOM**: 드래그 조작이 필요해서 포인터 이벤트를 받아야 하므로 DOM으로 렌더링.

| | 렌더링 방식 | 이유 |
|---|---|---|
| 도형 | DOM | 인터랙티브 (텍스트 편집, 임베드) |
| 선택 윤곽선 | Canvas 2D | 비인터랙티브, 일괄 렌더링이 빠름 |
| 기즈모 (핸들) | DOM | 드래그 조작 필요, 이벤트 처리 |

### CSS Containment로 리플로우 격리

3단계 containment 계층으로 리플로우 전파를 차단한다:

> 📍 packages/editor/editor.css

```css
.tl-canvas       { contain: strict; }            /* 최상위 — 완전 격리 (size+layout+style+paint) */
.tl-html-layer   { contain: layout style size; }  /* 레이어 간 전파 차단 */
.tl-shape        { contain: size layout; }         /* 도형 간 전파 차단 */
```

**`size`**: "내부 콘텐츠가 바뀌어도 내 크기는 안 변함" → 부모/형제 레이아웃 재계산 차단
**`layout`**: "내부 레이아웃이 외부에 영향 안 줌" → 자식 변경이 부모로 전파 차단

개별 도형에 `strict`가 아닌 `size layout`만 거는 이유: 도형은 `overflow: visible`이라 화살표/그림자가 경계 밖으로 나갈 수 있음. `paint` containment을 걸면 잘린다.

**`.tl-html-layer`가 1px × 1px인 이유**: 모든 도형이 `position: absolute` + `transform: matrix()`로 배치되므로 부모 크기와 무관. 1px로 고정하면 `contain: size`와 결합되어 자식이 아무리 바뀌어도 레이어 크기 계산 자체가 사라진다.

`contain`은 브라우저가 자동으로 해줄 수 없다. DOM만 보고는 "자식이 부모 크기에 영향을 주는지" 판단할 수 없으므로, 개발자가 명시적으로 **"내가 보장하니까 확인 안 해도 돼"**라고 선언하는 것.

### Culling — R-tree 기반 뷰포트 밖 도형 렌더 스킵

**RBush**(R-tree JS 구현) 라이브러리로 모든 도형의 바운딩 박스를 공간 인덱싱하고, 뷰포트 영역 쿼리로 O(log n)에 보이는 도형을 찾는다.

> 📍 packages/editor/src/lib/editor/managers/SpatialIndexManager/SpatialIndexManager.ts
> 📍 packages/editor/src/lib/editor/derivations/notVisibleShapes.ts

```
카메라 이동 → getViewportPageBounds() 갱신
  → rbush.search(viewportBounds)  // O(log n) 공간 쿼리
  → 뷰포트 밖 + canCull()인 도형 → display: none
```

바운딩 박스는 DOM에서 읽지 않고 **tldraw 데이터에서 수학적으로 계산**한다. 각 ShapeUtil의 `getGeometry()`가 도형의 좌표/크기 데이터로 바운딩 박스를 산출. 텍스트만 예외적으로 숨겨진 측정용 `<div>`에 `getBoundingClientRect()`를 호출하되, 데이터 변경 시 한 번만 측정하고 캐싱한다.

> 📍 packages/editor/src/lib/editor/managers/TextManager/TextManager.ts

culled 도형은 DOM에서 제거(unmount)가 아닌 `display: none`으로 숨김. 마운트/언마운트보다 가벼운 연산. 선택 중이거나 편집 중인 도형은 culling에서 제외된다.

### DOM 방식의 트레이드오프

| | DOM (tldraw) | Canvas API (Figma 등) |
|---|---|---|
| 도형 수 | 수백~수천 개 OK | 수만 개도 OK |
| 도형 안 인터랙션 | 네이티브 (텍스트 편집, 비디오, 폼) | 전부 직접 구현 |
| 커스텀 도형 | JSX 작성 | draw 함수 작성 |
| 히트 테스트 | 브라우저 이벤트 버블링 | 직접 기하 계산 |
| 텍스트 편집 | `contenteditable` | IME, 커서, 선택 전부 직접 구현 |
| 텍스트 구부림 | SVG `<textPath>` 가능하나 편집과 양립 어려움 | 자유롭지만 역시 편집은 직접 구현 |
| 적합한 유스케이스 | 화이트보드, 다이어그램 | 지도, 게임, 정밀 디자인 |

도형 수가 적고 인터랙티브 콘텐츠가 많으면 DOM이 유리하고, 도형이 수만 개이거나 픽셀 정밀 제어가 필요하면 Canvas API가 유리하다. DOM 방식이 성립하는 핵심 조건: CSS transform GPU 가속, `contain: size layout`로 리플로우 격리, 뷰포트 밖 도형 culling.

### GPU 가속 전략 — 힌트만 제공, 강제 안 함

`will-change: transform`이나 `translateZ(0)` 같은 확정 트리거를 의도적으로 안 쓴다. 대신 `contain: strict` + `content-visibility: auto`로 **브라우저가 알아서 판단**하도록 맡긴다.

> 📍 packages/editor/editor.css — `.tl-canvas`

```css
.tl-canvas {
    contain: strict;
    content-visibility: auto;
}
```

**이유**: 도형 1000개에 각각 GPU 레이어를 만들면 레이어당 텍스처 메모리(VRAM) 할당 + 매 프레임 합성 비용이 폭발한다. `.tl-canvas`(최상위)만 GPU 레이어 후보로 두고, 개별 도형은 일반 DOM으로 둬서 레이어 수를 최소화.

GPU를 적극적으로 써서 빠르게 그리는 전략이 아니라, **CPU 쪽에서 할 일을 최소화**(culling, contain, React 우회)해서 GPU 가속 없이도 충분하게 만든 구조.

> 📍 관련: `.study/07-performance.md` — useQuickReactor, culling 성능
> 📍 관련: `.study/02-ui-architecture.md` — ShapeUtil.component() 패턴
