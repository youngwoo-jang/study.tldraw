# 리소스 & 에셋 관리

## prebuild — 패키지 에셋 사전 빌드

모노레포 하위 패키지의 CSS, 아이콘 번들, 폰트, 번역 파일 등을 앱이 import할 수 있는 형태로 가공하는 단계.

> 📍 `yarn lazy prebuild` (lazyrepo가 변경분만 증분 빌드)

```
앱 빌드 전에 반드시 실행해야 함:
1. prebuild  → 패키지 에셋 준비
2. build-app → dotcom 앱 빌드 (prebuild 결과물을 import)
```

왜 "빌드"가 필요한가 — 원본 소스가 앱에서 바로 쓸 수 있는 형태가 아니기 때문:

- **아이콘**: SVG 수백 개 → 하나의 JS 모듈로 번들링 (`import { ArrowIcon } from '@tldraw/assets'`)
- **폰트**: 원본 → base64/최적화 포맷으로 CSS에 임베드 (외부 URL 로드 시 FOUT 방지)
- **번역**: 언어별 JSON → 타입 체크 + 누락 키 검증 + import 가능 모듈
- **CSS**: 각 패키지 CSS → PostCSS 처리

Next.js 같은 프레임워크는 이런 처리가 내장되어 있지만, tldraw는 SDK(라이브러리)라서 특정 프레임워크에 의존할 수 없어 직접 구축한다. 개발 시에는 파일을 잘게 나눠 관리하기 쉽게, 배포 시에는 합쳐서 성능 좋게 — 이 간극을 prebuild가 메워준다.

## R2 에셋 병합 (Asset Coalescing)

SPA 배포 시 이전 빌드의 에셋을 새 배포에 포함시켜, 기존 사용자의 구 버전 파일 요청이 404가 되지 않게 하는 기법.

> 📍 internal/scripts/deploy-dotcom.ts — `coalesceWithPreviousAssets()`
> 📍 관련 개념: `.study/00-dev-concepts.md` — 에셋 병합

### 문제 상황

Vite 빌드는 파일명에 해시 포함: `index-a3f2b1.js`. 새 배포하면 `index-x8k9d2.js`로 바뀌는데, 이미 앱을 열어놓은 사용자 브라우저는 여전히 `index-a3f2b1.js`를 lazy-load로 요청할 수 있다.

일반 웹앱은 CDN 캐시로 충분하지만, tldraw는 무한 캔버스라 **탭을 며칠씩 열어둔다**. CDN 캐시 만료(보통 수 시간) 후에도 구 에셋이 필요.

### 동작 방식

```
1. 현재 빌드 에셋을 R2에 tar.gz로 백업
2. R2에서 이전 1개월치 에셋 다운로드
3. 현재 빌드 폴더에 병합 (keep-existing: 새 파일 우선)
4. 병합된 결과를 통째로 Vercel에 배포
```

```typescript
// R2에 현재 에셋 백업
const objectKey = `${previewId ?? env.TLDRAW_ENV}/${new Date().toISOString()}+${sha}.tar.gz`
await new Upload({ client: R2, params: { Bucket: R2_BUCKET, Key: objectKey, Body } }).done()

// 이전 에셋 다운로드 + 병합 (keep-existing으로 새 파일 우선)
const out = tar.x({ cwd: assetsDir, 'keep-existing': true })
```

`keep-existing`이 중요한 이유: 같은 이름의 파일이 있으면 새 버전을 유지한다. Sentry debugId가 다르기 때문에 구 에셋으로 덮어쓰면 소스맵 매칭이 깨진다.

### 대부분의 앱에선 불필요

| 상황 | 에셋 병합 필요? |
|---|---|
| 일반 웹앱 (페이지 이동) | 불필요 — 자연스럽게 새 버전 로드 |
| SPA + 짧은 세션 | 불필요 — CDN 캐시로 충분 |
| SPA + 장시간 세션 + 잦은 배포 | **필요** — Figma, Google Docs, tldraw 등 |
