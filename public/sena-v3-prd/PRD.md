# sena-ai v3 — Product Requirements Document

**상태:** Draft (2026-05-09)
**작성:** 세나 (Channy 의뢰)
**관련 채널:** #project-sena (`C0AFW5Y133J`)
**기존 리포:** `Variel/sena` → 신규 `unlimiting-studio/sena-ai` `v3` orphan branch
**대체 대상:** `@sena-ai/*` 모노레포(core / cli / hooks / runtime-claude / runtime-codex / connector-slack / tools-slack / slack)

---

## 1. 배경 — 왜 v3가 필요한가

v2는 codex CLI와 Claude Agent SDK를 워커 프로세스로 직접 fork하고, Slack 통신을 자체 패키지(`@sena-ai/connector-slack`, `@sena-ai/slack-mrkdwn`, `@sena-ai/tools-slack`)로 처리해 왔다. 이 구조 위에서 안정적으로 sena/lumie/sooki/bren 네 에이전트가 운영되고 있지만, 최근 6주 채널 메모를 분류해 보면 디버깅 시간의 큰 덩어리가 다음 네 영역에 몰린다.

1. **Slack 디테일** — mrkdwn 이중 꺾쇠, `parse:'none'`과 `unfurl_links`의 양립 불가, `chat.startStream` 기반 streaming 메시지의 unfurl 비활성화, archive permalink labeling, 1.5초 throttle trailing flush, app_mention/message dedup race.
2. **워커 / IPC / restart** — `restart_agent` IPC 채널 race condition, drain 타임아웃, self-restart 데드락.
3. **모노레포 publish 운영** — `workspace:*`가 publish 시점에 exact pin으로 굳는 특성 때문에 core 수정 시 cli 동반 bump가 강제, 마이너 버전 통일 규칙, pnpm publish 강제.
4. **다중 에이전트 동기화** — 한 패키지를 수정하면 4개 에이전트(또는 6개) 전부 publish → install → restart 체인을 돌려야 한다.

(2)·(3)·(4)는 sena 자체가 짠 추상화 비용이고, (1)은 우리 Slack 구현이 Slack의 고유 동작에 충분히 잘 맞춰지지 못해서 생긴 비용이다.

community 생태계(특히 Vercel ai-sdk와 chat-sdk, Ben Vargas의 ai-sdk-provider-claude-code/codex-cli)가 이 덩어리들을 대부분 흡수해 줄 수 있다는 게 v3의 출발점이다. v3는 sena의 정체성("codex CLI + Claude Code를 에이전트 하네스로 쓰는 엔진")을 그대로 두면서, 그 위에서 **우리가 직접 짜는 코드의 표면적을 줄이는 것**을 목표로 한다.

## 2. 목적 / 비전

> **codex CLI와 Claude Code를 핵심 LLM 엔진으로 쓰는 단일 프로세스 에이전트 하네스를, ai-sdk 와 chat-sdk 위에 운영 보조 인프라(cron · hook · channel context · MCP · multi-connector)만 얹어 만든다.**

핵심 의존성은 외부 라이브러리에 맡기고, 우리는 운영 정책과 멀티 에이전트 배포 패턴만 책임진다.

## 3. 사용자

- **1차 사용자:** unlimiting-studio 내부에서 운영 중인 에이전트 — sena, lumie, sooki, bren. 각 에이전트의 운영자(Channy 본인 + 협업 인간)와 v3 위에서 새 에이전트를 만들 후속 빌더.
- **2차 사용자:** v2 모노레포 오픈소스 준비 흐름(2026-04-06)을 잇는 외부 빌더. 자기 codex/Claude Code 워크플로우를 Slack 봇으로 노출하고 싶은 개인/팀.

## 4. 핵심 시나리오

1. **Slack 트리거 응답.** 사용자가 채널에서 봇을 멘션하거나 봇이 참여 중인 스레드에 메시지를 남기면, 에이전트가 채널 컨텍스트와 메모리를 읽고 codex 또는 Claude Code 한 턴을 돌려 응답한다. 응답은 streaming preview로 점진 표시되며 최종 메시지는 unfurl·table·mrkdwn이 보존된다.
2. **정기 자동 작업.** 운영자가 cron 표현으로 정의한 스케줄(하트비트, 모닝 브리핑, 회고 등)이 시각마다 발화하여 에이전트의 한 턴을 비대화 트리거로 실행한다. prompt는 inline 문자열 또는 외부 파일 참조 둘 다 지원.
3. **외부 시스템 연동.** Sunny Gateway 같은 외부 시스템의 WebSocket relay에서 들어오는 turn 요청을 받아, 같은 에이전트 인스턴스가 일반 Slack 응답과 동일하게 처리한다. 시스템마다 별도 system prompt 합성이 가능하다.
4. **다중 connector 동시 운영.** 한 에이전트 프로세스 안에서 Slack과 Sunny가 동시에 살아 있고, 각자 독립된 trigger 규칙·system prompt·세션 ID를 갖는다.
5. **MCP 기반 도구 통합.** Claude Code의 `__native__` MCP, 사용자 정의 MCP 서버, 그리고 Zod 기반 inline tool(우리가 정의하는 함수)이 모두 한 turn 안에서 호출 가능하다.

## 5. 기능 요구사항 (FR)

| ID    | 영역                  | 요구사항                                                                                                                                                                                |
| ----- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-1  | LLM provider          | `ai-sdk-provider-claude-code` 와 `ai-sdk-provider-codex-cli`를 통해 두 엔진을 ai-sdk LanguageModel로 노출. 모델 동적 지정(`gpt-5.5` 같은 신규 모델명) 가능.                                |
| FR-2  | reasoning             | claude-code provider의 reasoning effort 옵션을 wrap하여 v2의 `reasoningEffort: low \| medium \| high \| max` 인터페이스를 보존.                                                            |
| FR-3  | Slack 통신            | `chat` + Slack 어댑터로 mention / thread / channel / reaction trigger를 모두 처리. trigger filter(`channelId`, `userId`, `text`, `threadTs`, `reaction` 기반)와 reaction abort 규칙 유지. |
| FR-4  | streaming             | Slack streaming preview + 최종 unfurl 복원이 v2 수준 이상으로 동작. mrkdwn 이중 꺾쇠 보존, archive permalink 자동 라벨링, table block 지원.                                              |
| FR-5  | Sunny Gateway         | chat-sdk 어댑터 인터페이스를 따르는 Sunny adapter를 신규 작성. 음성 응답 프로토콜(`turn.metrics`, binary frame 등)을 보존.                                                                |
| FR-6  | cron / heartbeat      | `cronSchedule()` 과 `heartbeat()` API 유지. prompt는 inline string 또는 `{ file: string }` 둘 다 지원.                                                                                    |
| FR-7  | hooks                 | TurnStart / TurnEnd / system 합성을 ai-sdk middleware (`wrapLanguageModel`) + chat-sdk middleware로 매핑. v2 hook 함수 시그니처는 마이그레이션 가이드와 함께 변경 가능.                  |
| FR-8  | channel context       | `channels.json` 과 채널별 `memory.md`를 turn 시작 시 자동 주입. 채널 ID 기반 라우팅.                                                                                                      |
| FR-9  | MCP                   | MCP 서버를 1급 시민으로 지원. v2의 `__native__` 패턴, owned localhost bridge, Stream closed 자동 reset 규칙은 유지하거나 그에 준하는 안정성 확보.                                          |
| FR-10 | inline tool           | `defineTool()` (Zod 스키마) 패턴을 MCP server 우회로 보존. 기존 v2 hook 코드의 마이그레이션 비용을 최소화.                                                                                  |
| FR-11 | session 영속성        | `.sessions.json` file session store 유지. 단일 프로세스 재시작 후에도 conversationId ↔ session 매핑이 살아남는다.                                                                          |
| FR-12 | multi-connector       | 한 에이전트 프로세스에서 Slack과 Sunny 어댑터가 동시에 active. 각 trigger마다 system prompt 다르게 주입 가능.                                                                              |

## 6. 비기능 요구사항 (NFR)

- **NFR-1 단일 프로세스.** worker fork / IPC / orchestrator-worker 분리 폐기. 단일 Node.js 프로세스로 실행하고, 외부 process manager(pm2 또는 systemd)가 lifecycle을 관리한다.
- **NFR-2 fail-fast.** 폴백 / 바이패스 금지(2026-04-14 차니 지시). 초기화 실패는 즉시 throw하고 프로세스가 죽는 게 정답이다.
- **NFR-3 외부 의존 우선.** 우리가 직접 짜는 코드는 사용자 정책(channel context, cron, hook, multi-connector orchestration)에 한정한다. Slack 프로토콜 디테일과 LLM 통신은 외부 라이브러리에 맡긴다.
- **NFR-4 오픈소스 양립.** 차니 머신에서 직접 운영되지만, 코드는 오픈소스 공개를 전제로 작성한다. 계정·키·내부 URL은 `wrangler.toml.example` 패턴(2026-04-06 학습)대로 분리.
- **NFR-5 운영 가시성.** v2의 `traceLogger` 수준 로그를 유지한다. 어떤 trigger가 어떤 turn을 일으켰고 어떤 tool이 호출됐는지 단일 stream으로 추적 가능해야 한다.

## 7. 범위 외 (v3 1차에서 빼는 것)

- **sena-platform** (Slack 앱 자동 프로비저닝, CF Workers + D1). 별도 차후 단계에서 다시 본다.
- **분산 멀티 호스트 운영.** v3는 단일 프로세스 가정.
- **Slack/Sunny 외 chat-sdk 어댑터(Discord, Teams, WhatsApp 등).** chat-sdk가 지원하지만 v3 1차에서는 라이브 connector 두 개만 검증한다.
- **자체 LLM API 호출 경로.** ai-sdk-provider 우회로 직접 OpenAI/Anthropic 호출하는 경로는 만들지 않는다 (codex CLI / Claude Code 의존성을 일관되게 유지).

## 8. 성공 기준

- **S-1** sena, lumie, sooki, bren 네 에이전트가 v3 위에서 정상 운영된다(현재 v2 기능을 100% 대체).
- **S-2** v3 마이그 후 6주 동안 채널 메모(#project-sena) 디버그 항목 빈도가 v2 직전 6주(2026-03-22~2026-05-03) 대비 절반 이하.
- **S-3** Slack 디테일 관련 디버그(parse / mrkdwn / streaming / unfurl)는 우리 코드 수정이 아니라 chat-sdk 어댑터 PR 또는 wrapper 패치로만 해결된다.
- **S-4** 외부 빌더 1명이 v3 위에 새 에이전트를 README만 보고 띄울 수 있다.

## 9. 잃는 것 / 검증 필요한 것 (Risk Matrix)

| 항목                                         | 영향                                          | 대응                                                                                          |
| -------------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| claude-code provider의 Zod tool 미지원       | v2 `defineTool()` 코드 손봐야 함              | MCP server 우회. inline-mcp-bridge 노하우로 어댑터 작성.                                      |
| claude-code provider의 reasoningEffort 미지원 | 직접 옵션 noop 됨                             | LanguageModel wrap으로 우리 옵션을 provider config에 떨어뜨림 (또는 upstream PR).              |
| codex-cli provider 모델명 사전 정의           | `gpt-5.5` 같은 동적 모델명 사용 불가          | upstream PR 보내거나 wrapper에서 모델 alias 직접 추가.                                        |
| chat-sdk Slack 어댑터의 디테일 커버 범위 미지 | mrkdwn / unfurl / streaming 회귀 가능         | 1차 마이그 직후 전수조사 1회 + 미커버 항목 wrapper 또는 upstream PR.                          |
| sena 고유 인프라(cron / hook / channel) 매핑 | ai-sdk middleware 패러다임에 1:1 매핑이 안 됨 | hook을 LanguageModel middleware 와 chat-sdk middleware 두 레이어로 나누는 설계 결정 별도 명시. |

## 10. 일정 (1차 가설)

- **5/9 (오늘)** PRD 확정, `unlimiting-studio/reports`에 호스팅.
- **5/10 ~ 5/13** sena-ai 새 `v3` orphan branch에 SPEC.md + 컴포넌트별 분할 스펙 작성.
- **5/14 ~ 5/20** 베어본 프레임워크 + sena_v2(비실행 상태) 또는 신규 PoC 에이전트 1개 마이그.
- **5/21 ~ 5/27** Slack 디테일 전수조사, 미커버 항목 패치, Sunny adapter 마이그.
- **5/28 ~ 6/10** 본 에이전트 순차 마이그 (sena → 브렌 → lumie → sooki).
- **6/11 ~ 6/24** v2 deprecation, 6주 디버그 항목 빈도 측정 시작.

## 11. 후속 의사결정 포인트

- 분담: 세나(이 문서 작성자)가 마이그 순서·잃는 항목·sena 고유 인프라 매핑 책임. 브렌이 ai-sdk-provider 코드 + chat-sdk Slack 어댑터 직접 까서 우리 기능 매트릭스 대비 커버 검증. (차니 confirm 대기)
- sena-platform의 v3 흡수 시점: 1차 마이그 완료 후 별도 라운드.
- v2 모노레포 history 보존 vs orphan branch 완전 reset: 차니 결정 필요.
