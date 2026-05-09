# sena-ai v3 — PoC 0단계 보고서

**상태:** Final (2026-05-10)
**작성:** 세나 (Channy 의뢰, 라이브 검증 완료)
**관련 채널:** #project-sena (`C0AFW5Y133J`)
**관련 문서:** [PRD rev. 2](/sena-v3-prd/) · [SPEC tree](https://github.com/unlimiting-studio/sena-ai/tree/v3)
**PoC 코드:** `~/agents/sena-poc/` (PRD-bound, npm 미공개)

---

## 한 줄 결론

`ai` + `chat` + `@chat-adapter/slack` + `@chat-adapter/state-pg` + `ai-sdk-provider-claude-code` 조합으로 **단일 Node 프로세스에서 Slack 멘션 한 turn이 끝까지 도는 것 + 영속화 + 동시성·인터럽트 제어가 라이브에서 검증됐다.** SPEC §0 미지수 8개 전부 닫힘. 본 마이그 §1 진입 GO.

---

## 검증된 미지수 (SPEC §0 체크리스트 8/8)

| # | 미지수 | 검증 방법 | 결과 |
|---|---|---|---|
| 1 | ai-sdk `LanguageModelV3Middleware` hook 가능 지점 | `transformParams` + `wrapStream`에 `traceLogger` 박고 chunk 분포 관찰 | ✅ `stream-start / response-metadata / reasoning-* / tool-input-* / tool-call / tool-result / text-* / finish` 전부 노출. 한 turn에 8 tool call까지 trace 잡힘 |
| 2 | chat-sdk 핸들러 hook 가능 지점 | `onNewMention` / `onSubscribedMessage` 라이브 트리거 | ✅ mention이 `onNewMention` → `subscribe()` → 후속 메시지 `onSubscribedMessage`로 정확히 라우팅 |
| 3 | `ScheduledMessage`가 `cronSchedule` 흡수 | (차니 정정: 다른 거. 흡수 안 함) | 우리 패턴으로 PoC: `setTimeout` → `chat.thread(threadId)` 외부 reference → `streamText` → `thread.post(string)`. 라이브 동작 ✅ |
| 4 | `@chat-adapter/slack` 출력 디테일 | (차니 결정: 별도 검증 불필요. 라이브 사용에서 markdown_text native·streaming·mrkdwn 자동 처리 확인) | ✅ |
| 5 | Zod inline tool 흡수 | (차니 결정: claude-code provider 미지원, MCP 우회 정답) | ✅ 결정 |
| 6 | chat-sdk observability 범위 | `traceLogger` middleware로 충분 — chat-sdk 자체 trace API 별도 의존 불필요 | ✅ |
| 7 | `@chat-adapter/state-pg` 실 동작 | Docker PG → `createPostgresState` → 자동 스키마(5 테이블) → `subscribe()` PG 영속 → 재시작 → 멘션 *없는* follow-up이 `onSubscribedMessage`로 라우팅 | ✅ **확정 결정 #2 검증 완료** |
| 8 | 프로세스 구조 입력 | 단일 프로세스 + drain wrapper + AbortController 기반 steering | ✅ **확정 결정 #3 권고안 도출** |

---

## 확정된 결정

### #2 — `@chat-adapter/state-pg` ✅ 채택

**라이브 검증:**
- Docker `postgres:16-alpine` 띄움 (15432 포트, dev용 password)
- `createPostgresState({ url, keyPrefix: "sena-poc" })` 한 줄 추가 → 부팅 시 5개 테이블 자동 생성: `chat_state_subscriptions`, `chat_state_locks`, `chat_state_queues`, `chat_state_cache`, `chat_state_lists`
- 멘션 → `await thread.subscribe()` → `chat_state_subscriptions`에 row 영속
- SIGTERM 종료 → 재시작 → 같은 thread에 *멘션 없는* 메시지 → `[turn.enter label=onSubscribedMessage]` ✅

**의미 (차니 의문에 답):**
codex/claude-code 자체 state(`~/.claude/projects/*.jsonl` 등)는 **모델 conversation context** 보관. chat-sdk state는 **thread routing/concurrency 메타데이터** 보관 (subscribe 영속 / lock / queue / per-thread state). 역할이 다르고 둘 다 필요. v2 `wasBotMentionedInThread()` 우회 패턴이 정확히 chat-sdk state 부재로 생긴 인메모리 한계였고, state-pg가 이걸 native 해결.

### #3 — 단일 프로세스 + 자체 drain wrapper + AbortController 기반 steering 레이어 ✅ 채택 권고

**검증된 동작:**
- **단일 Node 프로세스** — v2의 부모/워커 분리(SIGUSR2 rolling restart) 대신 한 프로세스에서 다중 connector·schedule·hot-reload 운영 가능
- **drain wrapper** — chat-sdk `Chat.shutdown()`은 어댑터/state disconnect만 하고 in-flight 핸들러는 기다리지 않음 (`node_modules/chat/dist/index.js:2454-2476` 직접 확인). 우리가 핸들러 위에 `inFlight` 카운터 + drain 루프(SIGTERM 받으면 inFlight=0 될 때까지 200ms 폴링, 60s timeout)를 박아서 graceful shutdown 보장
- **multi-socket coexist** — 같은 xapp 토큰으로 두 인스턴스 동시 socket 연결 가능 (Slack 공식 멀티 소켓 분산 라우팅) → zero-downtime rolling restart 패턴 가능
- **AbortController 기반 steering** — chat-sdk `concurrency: "concurrent"`로 thread lock 우회 → thread별 `Map<threadKey, AbortController>` 보관 → 새 메시지 도착 시 즉시 abort → ai-sdk `streamText({ abortSignal })`로 전파되어 stream 정상 종료(`error=1`로 마감) → 같은 핸들러 내부 loop가 새 컨텍스트로 다음 turn 시작
- **step-단위 steering(옵셔널)** — `tool-result` chunk를 step 경계 신호로 써서 mid-tool-call 인터럽트 회피. `SENA_POC_STEER_MODE=step`로 토글

**concurrency 4전략 라이브 검증:**
- `drop` (chat-sdk 기본): 새 메시지 무시 — 차니가 처음 본 "두 번째 멘션 드롭" 원인
- `queue`: 큐잉 + drainQueue 가장 최신 dispatch + `context.skipped`로 중간 메시지 핸들러 전달
- `concurrent` + 우리 즉시 abort steering: 진행 중 turn 즉시 중단 + 새 컨텍스트 재시작
- `concurrent` + 우리 step-steering: tool-result 경계까지 대기 후 abort

---

## 부수 발견 (본 마이그 시 wrapper / upstream PR 후보)

세 건 모두 chat-sdk(`chat@4.28.1` / `@chat-adapter/slack@4.28.1`)에서 발견. PoC PR이나 본 마이그 시점에 wrapper 패치 또는 upstream 이슈 등록 결정.

### 1. `Thread.handleStream`이 외부 reference에서 깨짐

`node_modules/chat/dist/index.js:1631`에서 `this._currentMessage.author.userId` dereference. `_currentMessage`가 null인 외부 트리거 thread (`chat.thread(threadId)`로 만든 reference)에서 stream post 호출 시 `Cannot read properties of undefined (reading 'userId')`로 깨짐. cron 발화처럼 incoming message가 없는 시나리오에서 streaming 출력 불가. 우리 PoC는 `await result.text` 후 string post로 우회.

### 2. abort 시 `chatStream.stop()`이 `not_authed` 던짐

진행 중 streamText abort 시 `@chat-adapter/slack/dist/index.js:3386`에서 stream close 처리 중 `not_authed` 1건. 새 turn에는 영향 없지만 abort된 stream의 클로즈 처리가 깨끗하지 않음.

### 3. `Chat.shutdown()` in-flight handler drain 부재

`Chat.shutdown()`(`chat/dist/index.js:2454`)이 어댑터/state disconnect만 하고 in-flight 핸들러는 추적/대기하지 않음. 우리가 drain wrapper로 메움.

---

## SPEC rev. 2로 흡수할 결과

| 분할 스펙 | 변경 사항 |
|---|---|
| `SPEC.md` | "검증 필요 영역" → "검증 완료" 섹션 / 확정 결정 #2 ✅ / #3 ✅ 단일 프로세스 + drain wrapper + steering / 부수 발견 3건 |
| `docs/specs/migration.md` | §0 체크리스트 결과 표 / 일정 갱신 (5/10 PoC 완료 → 5/14 본 마이그 §1) / "잃는 것" 섹션에 부수 발견 3건 추가 |
| `docs/specs/hooks.md` | `traceLogger` middleware 구체 패턴 (transformParams + wrapStream chunk count) |
| `docs/specs/connectors/slack.md` | state-pg 채택 / multi-socket coexist / 부수 발견 3건 |
| `docs/specs/architecture.md` | 단일 프로세스 + drain wrapper + steering 레이어 흐름도 |
| `docs/specs/schedules.md` | `ScheduledMessage` ≠ `cronSchedule` 명시 / 우리 패턴 (외부 cron + `chat.thread()` reference + `thread.post(string)`) |
| `docs/specs/tools.md` | Zod inline tool 미지원 → MCP 우회 확정 |

---

## 본 마이그 진입 신호

**GO 조건 충족:**
1. ✅ 8개 미지수 닫힘
2. ✅ 확정 결정 #2 / #3 권고안 확보
3. ✅ 부수 발견 3건 명시 (수용 가능 범위)
4. ✅ 본 마이그 일정 흐름 그대로 (5/14 §1 시작)

**§1에서 첫 작업으로 처리할 항목:**
- `~/agents/sena-poc/`를 sena_v2 또는 신규 베어본 에이전트로 승격 (SPEC `migration.md §1`)
- `defineConfig({ model, adapters, middlewares, schedules, state, mcpServers })` 1차 가설 그대로 구현
- 부수 발견 #1 (외부 reference stream post) wrapper 또는 upstream PR

---

## PoC 환경 (재현용)

**디렉토리:** `~/agents/sena-poc/`

**의존 (확정 버전):**
- `ai@6.0.177`
- `chat@4.28.1`
- `@chat-adapter/slack@4.28.1`
- `@chat-adapter/state-memory@4.28.1`
- `@chat-adapter/state-pg@4.28.1`
- `ai-sdk-provider-claude-code@3.4.4`
- `ai-sdk-provider-codex-cli@1.1.0`

**모드 토글 (env):**
- `SENA_POC_ENGINE=codex` — 기본 claude-code → codex-cli
- `SENA_POC_STEER_MODE=step` — 즉시 abort → 스텝 경계 abort
- `SENA_POC_STEER_MODE=queue` — chat-sdk lock + drainQueue (회귀 비교용)
- `DATABASE_URL=postgres://...` — state-memory → state-pg 토글
- `SENA_POC_CRON_TARGET=<threadId>` — 부팅 30초 후 cron 셀프 트리거 데모

**기본 동작 (env 없음):**
- 엔진: claude-code
- 인터럽트: 즉시 abort steering
- state: 인메모리
- cron: off

**라이브 봇:** `lily` (Slack User ID `U0APLTFB3E0`)

---

## AC (수용 기준)

1. ✅ Slack 멘션 한 turn이 chat-sdk · ai-sdk middleware · state-pg · drain wrapper 모두 통과해서 라이브 응답까지 도착
2. ✅ 재시작 후 subscription 영속으로 follow-up 메시지가 `onSubscribedMessage`로 라우팅
3. ✅ 진행 중 turn에 새 메시지 도착 시 즉시 abort + 새 컨텍스트로 재시작 (steering)
4. ✅ cron 셀프 트리거 turn이 일반 mention turn과 동일한 trace middleware 경로로 동작

---

## 검증 일자

2026-05-10 (KST). 라이브 검증 thread: `#project-sena` (`1778332211.565639`).
