# sena-ai v3 — Product Requirements Document

**상태:** Draft (2026-05-09, rev. 2)
**작성:** 세나 (Channy 의뢰)
**관련 채널:** #project-sena (`C0AFW5Y133J`)
**기존 리포:** `Variel/sena` → 신규 `unlimiting-studio/sena-ai` `v3` orphan branch
**대체 대상:** `@sena-ai/*` 모노레포(core / cli / hooks / runtime-claude / runtime-codex / connector-slack / tools-slack / slack)

---

## 1. 배경 — 왜 v3가 필요한가

v2는 codex CLI와 Claude Agent SDK를 워커 프로세스로 직접 fork하고, Slack 통신을 자체 패키지(`@sena-ai/connector-slack`, `@sena-ai/slack-mrkdwn`, `@sena-ai/tools-slack`)로 처리해 왔다. 이 구조 위에서 sena, lumie, sooki, 브렌 네 에이전트가 안정적으로 운영되고 있다. 모노레포 publish 운영(마이너 버전 통일, cli 동반 bump, pnpm 강제)이나 다중 에이전트 동기화는 정착된 규약이고 큰 부담이 아니다.

문제는 다른 두 영역에 시간이 몰린다는 점이다.

1. **Slack 디테일.** mrkdwn 이중 꺾쇠, `parse:'none'`과 `unfurl_links`의 양립 불가, `chat.startStream` 기반 streaming 메시지의 unfurl 비활성화, archive permalink labeling, 1.5초 throttle trailing flush, `app_mention`/`message` dedup race. 우리 Slack 구현이 Slack의 고유 동작을 충분히 잘 맞추지 못해서 생긴 비용이다. 핵심은 **"업계 표준 경험에 맞춰진 라이브러리"** 를 쓰면 우리가 이 디테일을 다시 짤 일이 없다는 것이다 (개별 버그는 `parse:'none'` 같은 우회를 들어내면 부분적으로 해결되지만, 그게 본질은 아니다).
2. **워커 / IPC / restart.** `restart_agent` IPC 채널 race condition, drain 타임아웃, self-restart 데드락. 우리가 직접 짠 orchestrator-worker 분리 위에서 발생하는 운영 비용. 단일 프로세스로 돌아가야 한다는 결론은 아직 없고, 더 좋은 방안은 검토 영역.

community 생태계 — Vercel ai-sdk와 chat-sdk, Ben Vargas의 `ai-sdk-provider-claude-code`/`codex-cli` — 가 위 두 덩어리를 흡수해 줄 수 있다는 게 v3의 출발점이다. v3는 sena의 정체성("codex CLI + Claude Code를 에이전트 하네스로 쓰는 엔진")을 그대로 두면서, 그 위에서 **우리가 직접 짜는 코드의 표면적을 줄이는 것**을 목표로 한다.

## 2. 목적 / 비전

> **codex CLI와 Claude Code를 핵심 LLM 엔진으로 쓰는 에이전트 하네스를, ai-sdk와 chat-sdk 위에 운영 보조 인프라(스케줄 · hook · channel context · MCP · multi-connector)만 얹어 만든다.**

핵심 의존성은 외부 라이브러리에 맡기고, 우리는 운영 정책과 멀티 에이전트 배포 패턴만 책임진다.

## 3. 사용자

- **1차 사용자:** unlimiting-studio 내부에서 운영 중인 에이전트 — sena, lumie, sooki, 브렌. 각 에이전트의 운영자(Channy 본인 + 협업 인간)와 v3 위에서 새 에이전트를 만들 후속 빌더.
- **2차 사용자:** v2 모노레포 오픈소스 준비 흐름(2026-04-06)을 잇는 외부 빌더. 자기 codex/Claude Code 워크플로우를 Slack 봇으로 노출하고 싶은 개인/팀.

## 4. 핵심 시나리오

1. **Slack 트리거 응답.** 사용자가 채널에서 봇을 멘션하거나 봇이 참여 중인 스레드에 메시지를 남기면, 에이전트가 채널 컨텍스트와 메모리를 읽고 codex 또는 Claude Code 한 턴을 돌려 응답한다. 응답은 streaming preview로 점진 표시되며 최종 메시지는 unfurl·table·mrkdwn이 chat-sdk Slack 어댑터의 표준 동작대로 노출된다.
2. **정기 자동 작업.** 운영자가 cron 표현으로 정의한 스케줄(모닝 브리핑, 회고, 주기적 점검 등)이 시각마다 발화하여 에이전트의 한 턴을 비대화 트리거로 실행한다.
3. **외부 시스템 연동.** Sunny Gateway 같은 외부 시스템의 WebSocket relay에서 들어오는 turn 요청을 같은 에이전트 인스턴스가 받아 처리한다 (구체 어댑터는 v3 1차 범위 외, 차후).
4. **다중 connector 동시 운영.** 한 에이전트 프로세스 안에서 여러 trigger 소스가 동시에 살아 있고, 각자 독립된 trigger 규칙·system prompt·세션을 갖는다.
5. **MCP 기반 도구 통합.** 사용자 정의 MCP 서버(stdio · http 모두)를 1급으로 붙여 도구 호출이 가능하다.

## 5. 기능 요구사항 (FR)

| ID    | 영역             | 요구사항                                                                                                                                                                                                                       |
| ----- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-1  | LLM provider     | `ai-sdk-provider-claude-code`와 `ai-sdk-provider-codex-cli`를 통해 두 엔진을 ai-sdk LanguageModel로 노출. 모델 / reasoning 설정은 cc·codex의 시스템 설정을 그대로 따른다.                                                       |
| FR-2  | Slack 통신       | `chat` + Slack 어댑터로 mention · thread · channel · reaction · button · slash · modal trigger를 처리. trigger 라우팅은 chat-sdk가 제공하는 핸들러 위에서 짠다.                                                                |
| FR-3  | Slack 출력       | mrkdwn / streaming / unfurl / table 등의 디테일은 **chat-sdk Slack 어댑터의 동작을 그대로 따른다.** 우리가 별도 정의하지 않는다.                                                                                                |
| FR-4  | 스케줄           | cron 표현 기반 정기 트리거(`cronSchedule`). chat-sdk가 `ScheduledMessage` 클래스를 가지고 있어 자체 흡수될 가능성 있음 — 1차 마이그 중 검증.                                                                                   |
| FR-5  | hook             | TurnStart / TurnEnd / system prompt 합성 의도를 ai-sdk middleware (`transformParams`, `wrapGenerate`, `wrapStream`) 와 chat-sdk 핸들러 위에 다시 짠다. **실제 hook 가능 지점과 함수 시그니처는 v2와 다를 수 있다 — 마이그 중 검증.** |
| FR-6  | channel context  | 채널 단위 컨텍스트 주입(설명 / 메모리 / 리포지토리 등)이 한 턴의 system prompt에 합성된다. 구현 형태는 ai-sdk와 chat-sdk 기능을 살펴보며 결정.                                                                                  |
| FR-7  | MCP              | 사용자 정의 MCP 서버(stdio · streamable HTTP 등)를 1급으로 붙여 도구 호출이 가능하다. v2에서 우리가 임시로 쓰던 `__native__` 같은 자체 컨벤션은 가져가지 않는다.                                                                |
| FR-8  | inline tool      | Zod 기반 inline tool은 chat-sdk / ai-sdk 자체 tool 메커니즘으로 흡수되는 게 1차 가설. 흡수 안 되는 영역은 MCP 서버 우회로 — 마이그 중 결정.                                                                                     |
| FR-9  | session 영속성   | **chat-sdk가 제공하는 state adapter** (`@chat-adapter/state-redis` / `state-ioredis` / `state-pg` / `state-memory`) 를 그대로 사용한다. 우리가 별도 session 영속성 코드를 만들지 않는다. **선택할 adapter는 차니 결정 큐.**         |
| FR-10 | multi-connector  | 한 에이전트 프로세스에서 여러 connector 어댑터가 동시에 active. trigger마다 system prompt 다르게 주입 가능.                                                                                                                    |

## 6. 비기능 요구사항 (NFR)

- **NFR-1 fail-fast.** 폴백 / 바이패스 금지(2026-04-14 차니 지시). 초기화 실패는 즉시 throw하고 프로세스가 죽는 게 정답.
- **NFR-2 외부 의존 우선.** 우리가 직접 짜는 코드는 사용자 정책(channel context, schedule, hook 합성, multi-connector orchestration)에 한정한다. Slack 프로토콜 디테일과 LLM 통신은 외부 라이브러리에 맡긴다.
- **NFR-3 오픈소스 양립.** 차니 머신에서 직접 운영되지만, 코드는 오픈소스 공개를 전제로 작성한다.
- **NFR-4 운영 가시성 (검토 필요).** v2의 `traceLogger` 수준의 turn 추적은 ai-sdk `wrapStream` middleware로 chunk·tool 호출 단위까지 직접 짤 수 있다. chat-sdk 쪽의 자체 observability(이벤트 로그, 핸들러 trace 등) 범위는 별도 검토 후 NFR로 확정. 두 레이어가 어느 쪽을 책임지는지가 명확해져야 한다.

## 7. 범위 외 (v3 1차에서 빼는 것)

- **sena-platform** (Slack 앱 자동 프로비저닝, CF Workers + D1). 별도 차후 단계에서 다시 본다.
- **Sunny Gateway 어댑터.** 시나리오 3에서 의도는 명시하지만, v3 1차 어댑터는 chat-sdk Slack 하나에 집중. Sunny 연동은 별도 라운드.
- **Slack 외 chat-sdk 어댑터(Discord, Teams, WhatsApp 등).** chat-sdk가 지원하지만 v3 1차에서는 Slack만 라이브 검증.
- **자체 LLM API 호출 경로.** ai-sdk-provider 우회로 직접 OpenAI/Anthropic 호출하는 경로는 만들지 않는다.

## 8. 성공 기준

- **S-1** sena, lumie, sooki, 브렌 네 에이전트가 v3 위에서 정상 운영된다(현재 v2 기능을 100% 대체).
- **S-2** v3 마이그 후 6주 동안 Slack 디테일 / 워커-IPC-restart 영역의 채널 메모(#project-sena) 디버그 항목 빈도가 v2 직전 6주(2026-03-22 ~ 2026-05-03) 대비 절반 이하.
- **S-3** Slack 디테일 회귀가 발생해도 우리 코드 수정이 아니라 chat-sdk 어댑터 PR 또는 wrapper 패치로만 해결된다.
- **S-4** 외부 빌더 1명이 v3 위에 새 에이전트를 README만 보고 띄울 수 있다.

## 9. 잃는 것 / 검증 필요한 것 (Risk Matrix)

| 항목                                                | 영향                                          | 대응                                                                                              |
| --------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| chat-sdk Slack 어댑터의 디테일 커버 범위 미지       | mrkdwn / unfurl / streaming 회귀 가능         | 1차 마이그 직후 전수조사 1회 + 미커버 항목 wrapper 또는 upstream PR.                              |
| ai-sdk middleware의 hook 가능 지점                  | v2 hook 함수 시그니처 그대로 못 옮김           | `transformParams`/`wrapGenerate`/`wrapStream` 위에 다시 설계. 마이그 중 결정.                      |
| chat-sdk session state adapter 선택                 | 운영 의존성 변화                               | 차니 결정 큐. in-memory 시 재시작 시 history 손실 / Redis·PG 시 외부 의존 추가.                     |
| chat-sdk `ScheduledMessage`가 cronSchedule 흡수 여부 | FR-4 구현 형태 결정                           | API doc 추가 검토 + 1차 마이그에서 시도.                                                          |
| Zod inline tool                                     | v2 `defineTool()` 코드 손봐야 함              | chat-sdk·ai-sdk 자체 tool 메커니즘 우선 확인. 안 되면 MCP server 우회.                             |
| chat-sdk의 자체 observability 범위                  | NFR-4 책임 분리 결정                           | 1차 마이그 중 chat-sdk 핸들러 trace 가능 여부 확인. 부족하면 ai-sdk middleware 쪽에서 보강.        |

## 10. 일정 (1차 가설)

- **5/9 (오늘)** PRD 확정, `unlimiting-studio/reports`에 호스팅.
- **5/10 ~ 5/13** sena-ai 새 `v3` orphan branch에 SPEC.md + 컴포넌트별 분할 스펙 작성.
- **5/14 ~ 5/20** 베어본 프레임워크 + sena_v2(비실행 상태) 또는 신규 PoC 에이전트 1개 마이그.
- **5/21 ~ 5/27** Slack 디테일 전수조사, 미커버 항목 패치.
- **5/28 ~ 6/10** 본 에이전트 순차 마이그 (sena → 브렌 → lumie → sooki).
- **6/11 ~ 6/24** v2 deprecation, 6주 디버그 항목 빈도 측정 시작.

## 11. 후속 의사결정 포인트

- **분담.** 세나(이 문서 작성자)가 마이그 순서·잃는 항목·sena 고유 인프라 매핑 책임. 브렌이 `ai-sdk-provider-*` 코드 + chat-sdk Slack 어댑터 직접 까서 우리 기능 매트릭스 대비 커버 검증. _(차니 confirm 대기)_
- **chat-sdk state adapter 선택.** in-memory / `@chat-adapter/state-pg` / `@chat-adapter/state-redis` 중. _(차니 결정 필요)_
- **프로세스 구조.** v2의 orchestrator-worker 분리를 유지할지, 단일 프로세스로 갈지, 더 좋은 방안을 찾을지. _(검토 후 차니 confirm)_
- **앱 자체 패키지명.** v3는 우리 모노레포가 사라지고 *앱 한 개*가 publish 대상이 된다. 이름은 `@unlimiting-studio/sena` / `@sena-ai/sena` / 다른 안 중. _(차니 결정 필요)_
- **sena-platform의 v3 흡수 시점.** 1차 마이그 완료 후 별도 라운드.
- **v2 모노레포 history 보존 vs orphan branch 완전 reset.** _(차니 결정 필요)_
