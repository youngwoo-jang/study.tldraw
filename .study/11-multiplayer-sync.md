# 멀티플레이어 & 동기화

## Zero 백엔드 (Rocicorp)

Rocicorp이 만든 외부 로컬-퍼스트 동기화 엔진. tldraw 자체 구현이 아니라 `@rocicorp/zero` 의존성으로 사용한다.

> 📍 apps/dotcom/zero-cache/ (Fly.io 배포 설정)
> 📍 internal/scripts/deploy-dotcom.ts — `deployZeroBackend()`
> 📍 관련 개념: `.study/00-dev-concepts.md` — 로컬-퍼스트

### 역할

PostgreSQL ↔ 클라이언트 사이의 동기화 레이어. 클라이언트는 로컬 데이터를 읽고 쓸 뿐이고, 서버 동기화는 Zero가 알아서 처리한다.

```
클라이언트 A (브라우저)          클라이언트 B (브라우저)
  로컬 캐시 (IndexedDB)          로컬 캐시 (IndexedDB)
       ↕                              ↕
    ┌──────────────── Zero 서버 ──────────────────┐
    │  View Syncer: 각 클라이언트에게 필요한       │
    │              데이터만 골라서 실시간 push      │
    │                                              │
    │  Replication Manager: PostgreSQL의            │
    │              변경사항을 감시하고 수집          │
    └──────────────────────────────────────────────┘
                        ↕
                   PostgreSQL
```

### Fly.io에 배포하는 이유

Zero는 DB 변경사항을 지속적으로 감시하면서 연결된 클라이언트에 실시간 push하는 **상주 프로세스**. Cloudflare Workers는 요청-응답 기반(최대 30초)이라 안 맞는다.

| 요구사항 | Cloudflare Workers | Fly.io |
|---|---|---|
| PostgreSQL TCP 직접 연결 | 제한적 | 가능 |
| 장기 실행 프로세스 | 요청당 최대 30초 | 항상 실행 |
| 메모리 상태 유지 | 요청 끝나면 소멸 | 프로세스 상주 |

### 배포 모드 (환경별)

```typescript
// deploy-dotcom.yml에서 환경별 결정
production → DEPLOY_ZERO=false       // 아직 production 미적용
staging    → DEPLOY_ZERO=flyio-multinode  // 멀티노드로 테스트 중
preview    → DEPLOY_ZERO=flyio       // 단일노드 (빠르고 저렴)
```

**단일 노드** (`flyio`): 하나의 프로세스가 replication + view sync 모두 처리. 프리뷰용.

**멀티 노드** (`flyio-multinode`): 역할 분리 + 스케일아웃. 스테이징에서 검증 중.

```
Replication Manager (1대) — DB 변경사항 수집
        ↓ 내부 네트워크 (http://{rm-app}.internal:4849)
View Syncer (2대) — 클라이언트에게 동기화된 뷰 제공
```

### tldraw 자체 동기화 엔진 (tlsync)과의 관계

tldraw에는 원래부터 자체 동기화 엔진(`packages/sync/`)이 있다. Zero는 비교적 최근 도입 중이며, production에서는 아직 tlsync를 사용한다. Zero는 staging에서 검증 단계.

## tlsync 동기화 프로토콜

tldraw가 자체 구현한 실시간 동기화 엔진. Yjs/Automerge 같은 기존 CRDT 라이브러리를 사용하지 않고 처음부터 만들었다.

> 📍 packages/sync-core/src/lib/protocol.ts — 프로토콜 정의 (v8)
> 📍 packages/sync-core/src/lib/TLSyncRoom.ts — 서버 측 방 관리
> 📍 packages/sync-core/src/lib/TLSyncClient.ts — 클라이언트 측 동기화
> 📍 packages/sync/src/useSync.ts — React 훅

### 왜 별도 서버가 필요한가

멀티플레이어(여러 사용자가 같은 캔버스를 동시 편집)를 위해 브라우저만으로는 해결 불가능한 4가지 문제:

1. **순서 보장** — 서버가 `serverClock`으로 변경의 인과적 순서를 결정
2. **충돌 해결** — 동시 수정 시 서버가 commit/discard/rebase를 판정
3. **접속자 추적 (Presence)** — 커서 위치, 선택 영역을 모든 참여자에게 브로드캐스트
4. **영속성** — 브라우저 종료 후에도 문서 생존

### 프로토콜 메시지 (v8)

| 메시지 | 방향 | 역할 |
|--------|------|------|
| `connect` | Server→Client | 초기 문서 상태 + 스키마 + `serverClock` 전달 |
| `push` | Client→Server | 로컬 변경(diff) + presence(커서) 전송 |
| `patch` | Server→Client | 다른 사용자의 변경사항 브로드캐스트 |
| `push_result` | Server→Client | commit(수락) / discard(폐기) / rebaseWithDiff(재조정) |
| `ping`/`pong` | 양방향 | Keep-alive |

### 서버 권위 모델 (Server-Authoritative)

순수 CRDT가 아닌 서버 중심 설계:

```
Client A ──push(diff)──→  Server (clock: 42 → 43)  ──patch──→ Client B
Client B ──push(diff)──→  Server (clock: 43 → 44)  ──patch──→ Client A
```

- 서버(`TLSyncRoom`)가 논리적 시계를 유지하며 변경 순서를 결정
- 레코드 단위 연산(Put/Patch/Delete)으로 도형 전체를 다루므로 텍스트 CRDT보다 단순
- CRDT 라이브러리를 안 쓴 이유: 도형은 레코드 통째로 덮어쓰기가 자연스럽고, 변경 이력을 보관하지 않아 데이터가 가볍고, 서버가 거부(discard)할 수 있어 권한/유효성 관리가 쉬움

### 전송 최적화

- 솔로 모드: **1 FPS** 배치 전송
- 협업 모드: **30 FPS** 배치 전송
- 자동 감지하여 전환

### 패키지 구조

```
packages/sync-core/  — 프로토콜, TLSyncRoom, TLSyncClient (플랫폼 무관)
packages/sync/       — React 훅(useSync), 브라우저 전용 어댑터
```

## Cloudflare 아키텍처

Worker(서버리스)가 게이트웨이 역할만 하고, Durable Object가 실제 동기화를 처리한다.

> 📍 apps/bemo-worker/src/worker.ts — Worker 라우터
> 📍 apps/bemo-worker/src/BemoDO.ts — Durable Object

### 요청 흐름

```
Client → Worker (itty-router)
           │
           │ /connect/:slug → slug을 DO ID로 변환
           ▼
         Durable Object (BemoDO)
           ├── this.room: TLSyncRoom (메모리 상주)
           ├── WebSocket × N (연결된 클라이언트)
           └── R2 (스냅샷 영속화, 5초 간격)
```

Worker는 URL 라우팅 + 에셋(이미지 업로드/다운로드)만 처리. WebSocket 관리, 상태, 충돌 해결은 전부 DO 내부에서 동작한다.

### Worker가 게이트웨이인 이유

```typescript
// worker.ts 핵심 (72~78줄)
.get('/connect/:slug', (request) => {
    const id = this.env.BEMO_DO.idFromName(slug)  // slug → DO ID
    return this.env.BEMO_DO.get(id).fetch(request) // DO에 위임
})
```

### Durable Objects를 선택한 이유

- **방당 단일 인스턴스 보장** — 전 세계에서 roomId당 정확히 하나. 락 없이 원자적 상태 관리
- **네트워크 홉 0** — 같은 방의 모든 연결이 하나의 프로세스 메모리에서 처리 (Redis 방식은 서버 간 Pub/Sub 필요)
- **WebSocket Hibernation** — 유휴 시 메모리 해제하되 연결은 유지, 비용 절감
- **소규모 다수 방에 최적** — 2~20명이 다수 존재하는 tldraw 구조에 적합 (수천 명 단일 채널에는 부적합)

### 영속성

| 환경 | 저장소 구성 |
|------|------------|
| bemo (데모) | 인메모리 + R2 스냅샷 (5초 간격) |
| 프로덕션 | Durable Object SQLite + R2 장기보관 + PostgreSQL 메타데이터 |

> 📍 기술스택 요약: `.study/01-dev-infra.md` — deploy-bemo.yml
> 📍 관련 개념: `.study/00-dev-concepts.md` — CRDT, 서버리스 vs Durable Objects
