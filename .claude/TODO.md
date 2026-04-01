# UI 아키텍처 스터디 TODO

## ShapeUtil 렌더링
- [ ] `ShapeUtil.component()` → React JSX 반환 패턴
- [ ] `<SVGContainer>` vs `<HTMLContainer>` 사용 구분
- [ ] 자유 그리기(DrawShapeUtil) — `svgInk()`로 SVG path 생성 파이프라인
- [ ] 압력 감지 → `setStrokePointRadii()` → 좌/우 윤곽 트랙 → 닫힌 path
- [ ] Geo 도형이 SVG(도형) + HTML(텍스트 라벨)을 동시에 쓰는 구조

## 폰트 시스템
- [ ] FontManager — 폰트 라이프사이클 관리 (로딩, 추적, 캐싱)
- [ ] 기본 4종 폰트 (draw/sans/serif/mono)와 CSS 변수 매핑
- [ ] `TLFontFace` 인터페이스로 커스텀 폰트 등록
- [ ] 리치 텍스트(TipTap) 기반 텍스트 편집과 폰트 감지
- [ ] SVG 내보내기 시 폰트 임베딩 (`FontEmbedder`)

## UI Context 시스템
- [ ] ActionsProvider — 100+ UI 액션 정의/디스패치 구조
- [ ] ToolsProvider — 도구 등록과 `useTools()` 패턴
- [ ] DialogsProvider — Radix UI 기반 모달 관리 (`addDialog`/`removeDialog`)
- [ ] ToastsProvider — 알림 시스템 (severity, 자동 닫힘)
- [ ] EventsProvider — 50+ UI 이벤트 타입과 소스 추적

## CSS 아키텍처
- [ ] `tlui-` 접두사 BEM 네이밍 컨벤션
- [ ] z-index 레이어링 시스템 (CSS 변수 `--tl-layer-*`)
- [ ] `tl-container` 루트 컨테이너와 포커스 관리
