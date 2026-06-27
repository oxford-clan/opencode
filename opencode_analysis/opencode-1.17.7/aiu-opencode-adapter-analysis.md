# aiu-opencode-adapter 프로젝트 분석

> 분석 대상: `D:\AREA51\workspace\aiu-opencode-adapter`
> 분석 일자: 2026-06-17
> 분석 기준: `docs/` 산출물 + `packages/` 구현 소스 + roadmap/HANDOFF 진행 기록

---

## 1. 한 줄 결론

`aiu-opencode-adapter`는 **opencode가 보내는 OpenAI Chat Completions 요청을, native tool call을 지원하지 않는 사내 AIU(MISO/Dify 계열) Workflow LLM 노드로 중계하면서, 단일 JSON 텍스트 출력을 OpenAI `tool_calls` 계약으로 역변환해 주는 어댑터 계층**이다. 핵심 가치는 "chat-only LLM에 native tool calling 능력을 얹는 변환기"에 있다.

---

## 2. 문제 정의와 목적

| 항목 | 내용 |
| --- | --- |
| opencode가 기대하는 것 | OpenAI-compatible provider 계약 (`GET /v1/models`, `POST /v1/chat/completions`, streaming SSE, `tools`/`tool_calls`/`usage`) |
| AIU Workflow가 제공하는 것 | `POST /ext/v1/workflows/run` (`inputs`/`mode`/`user`), chat-only LLM 노드, **native tool calling 미지원** |
| 간극 | opencode의 tool 기반 agent 루프 ↔ workflow의 텍스트 입출력 |
| adapter의 역할 | 두 계약 사이의 **양방향 변환 계층** (요청 정규화 + 응답 역정규화 + 추적/마스킹/에러 변환) |

가장 어려운 설계 과제는 **"chat 모드 LLM에 tool call을 시킨다"**는 점이다. 이를 위해:
1. workflow LLM 노드 System 프롬프트가 LLM에게 **항상 정확히 하나의 JSON 객체**(`{"answer":..., "tool_calls":...}`)만 출력하도록 강제하고,
2. adapter가 그 JSON 텍스트를 파싱해 OpenAI `tool_calls` / `content` 로 분기 변환한다.

즉 native tool calling은 **프롬프트 엔지니어링 + adapter 파싱**의 조합으로 에뮬레이션된다.

---

## 3. 저장소 구조

```
aiu-opencode-adapter/
├─ docs/                         # 설계자(설계 전용) 작업 영역
│  ├─ PRD.md                     # 목적/범위/성공 기준
│  ├─ spec/                      # 인터페이스 계약 (행동 규칙)
│  │  ├─ opencode-interface-spec.md       # opencode ↔ adapter 계약
│  │  ├─ adapter-functional-spec.md       # adapter 내부 행동 규칙
│  │  ├─ workflow-interface-spec.md       # adapter ↔ workflow 계약
│  │  ├─ workflow-llm-system-prompt-spec.md  # ★ tool-call 에뮬레이션 프롬프트
│  │  ├─ workflow-sandbox-spec.md
│  │  └─ logging-spec.md
│  ├─ task/                      # 실행 단위 작업 지시서 + roadmap
│  ├─ adr/                       # 설계 결정 기록 (adr-002~006)
│  ├─ context/                   # 외부 API 명세/참고 자료
│  └─ run/                       # 검증 런북, 캡처 payload, .http 스모크
├─ packages/
│  ├─ adapter/                   # ★ 변환 코어 (TypeScript, esbuild 번들)
│  ├─ workflow-sandbox/          # 개발용 가짜 workflow upstream (mock/llm 모드)
│  └─ adapter-log/               # 웹 기반 로그 뷰어 (실시간 모니터링)
└─ package.json                  # pnpm workspace, dev:adapter/dev:sandbox/dev:log
```

### 역할 기반 워커 모델 (특이점)

이 프로젝트는 사람/에이전트의 역할을 **디렉터리 경계로 강제**한다:

| 역할 | 작업 영역 | Task ID 접두어 |
| --- | --- | --- |
| 설계자(docs) | `docs/`만 수정 | `DOC-xxx` |
| adapter 개발자 | `packages/adapter/` | `ADP-xxx` |
| sandbox 개발자 | `packages/workflow-sandbox/` | `SBX-xxx` |
| workflow 작업자(휴먼) | 실제 workflow 엔진 설정 | `WFL-xxx` |
| 로그 도구 개발자 | `packages/adapter-log/` | `LOG-xxx` |

> **계약 우선(contract-first) 원칙**: 불명확한 요구는 코드로 추측하지 않고 `task` 또는 `ADR`로 되돌린다. 구현보다 계약과 성공 기준을 먼저 고정한다. 문서는 한국어, API 필드/path/model id는 영어 원문 유지.

---

## 4. 아키텍처 & 데이터 흐름

```
┌──────────┐  OpenAI Chat Completions   ┌───────────┐  /ext/v1/workflows/run   ┌──────────────────┐
│ opencode │ ─────────────────────────▶ │  adapter  │ ───────────────────────▶ │ AIU Workflow      │
│ (TUI/CLI)│   POST /v1/chat/completions│           │   inputs/mode/user       │  (또는 sandbox)    │
│          │ ◀───────────────────────── │           │ ◀─────────────────────── │  chat-only LLM 노드 │
└──────────┘  SSE / completion + usage  └───────────┘  SSE / CompletionResponse └──────────────────┘
                                              │
                                              ▼  (stdout/파일 로그)
                                        ┌───────────┐
                                        │ adapter-log│  웹 뷰어로 request 단위 추적
                                        └───────────┘
```

### 요청 경로 (opencode → workflow)

`packages/adapter/src/workflow-client.ts`의 `buildWorkflowInputs()`가 핵심:

1. `messages`에서 `role:"system"`을 분리.
2. system 프롬프트에서 정규식으로 `<env>`, `<available_skills>` 블록을 추출 → `opencode_env`, `opencode_available_skills` 파라미터로 전송 (`extractOpenCodeBlocks()`).
   - opencode의 거대한 정적 system prompt 전문을 그대로 보내면 사내 workflow의 콘텐츠 정책 필터(jailbreak 패턴 유사)에 걸리는 문제가 있어, **동적 부분만 태그 단위로 추출**하는 방식으로 진화함 (ADP-015→016→017→023의 역사).
3. system을 제외한 **전체 대화 이력**(user/assistant/tool 인터리빙 순서 보존)을 `messages`로 `JSON.stringify`해서 전송.
   - role별로 필터링하면 멀티턴 tool follow-up에서 순서가 깨져 무한 루프가 발생하던 버그를 해결한 결과 (ADP-022→026→029).
4. `tools`, `tool_choice`를 직렬화해 전달.

> AIU Workflow Start 노드가 Object/Array 타입을 잘 지원하지 않아, 복합 입력은 모두 **JSON 문자열(String)로 직렬화**해 전달하는 제약이 반복적으로 드러남.

### 응답 경로 (workflow → opencode) — tool-call 에뮬레이션의 핵심

`classifyOutput()` → `parseMaybeJson()`이 LLM의 텍스트 출력을 분류:

| LLM 출력 형태 | adapter 판단 | opencode로 변환 |
| --- | --- | --- |
| `{"tool_calls":[...], "answer":...}` | 구조화 tool call | `message.tool_calls` + `finish_reason:"tool_calls"` |
| `{"answer":"...", "tool_calls":null}` | 텍스트 | `message.content` + `finish_reason:"stop"` |
| 순수 문자열 | 텍스트 | `message.content` |
| 설명문 + JSON 혼합 | 비준수 → balanced-brace 스캔으로 JSON 복구, prefix는 answer로 승격 | warn 로그 + best-effort |
| 깨진/미완성 JSON | `malformed_json_output` warn | 텍스트 fallback |

- `function.arguments`는 LLM이 plain object로 출력 → adapter가 OpenAI 계약에 맞게 **JSON string으로 재직렬화** (`classifyOutput` 내부).
- Gemini의 `thought_signature` 등 미지 필드는 `extra_content`로 **투명 통과**시킴 (ADP-024/SBX-020).

### Streaming 변환

workflow SSE 이벤트(`workflow_started` / `text_chunk` / `workflow_finished`)를 OpenAI SSE chunk로 매핑. 텍스트를 누적했다가 `workflow_finished` 시점에 `classifyOutput()`을 호출해 tool_calls/text를 분기하고, 최종 chunk에 `usage` + `finish_reason`을 실어 `[DONE]`으로 종료.

---

## 5. 핵심 구현 메커니즘 (adapter)

| 메커니즘 | 위치 | 설명 |
| --- | --- | --- |
| 요청 분기 | `adapter.ts:handler` | `GET /v1/models`, `POST /v1/chat/completions` 라우팅. (title/agent 구분 로직은 ADP-018에서 dead code로 제거됨 — opencode는 항상 tools 전송) |
| tool-call 파싱 | `workflow-client.ts:classifyOutput/parseMaybeJson` | JSON 텍스트 → tool_calls/text 분기 + 비준수 응답 복구 |
| usage 계산 | `workflow-client.ts:buildUsage` | **문자수 기반**: `prompt_tokens`=system 제외 messages JSON 길이, `completion_tokens`=응답 길이. opencode context bar(`limit.context`/`limit.output`)와 연동하기 위한 근사치 (실제 workflow 토큰과는 단위가 다름) |
| 루프 감지 | `adapter.ts:detectRepeatedToolCall` | 인접 2개 assistant 턴의 `name::arguments` fingerprint 비교. write/edit 반복 시 workflow 호출 없이 synthetic 200 응답 반환 (ADP-040/044) |
| 타임아웃/취소/재시도 | `workflow-client.ts` | 기본 5분 타임아웃, AbortSignal 결합(client disconnect + timeout), 502/503 제한적 재시도 |
| 디버그 덤프 | `workflow-client.ts:writeDump` | `ADAPTER_DEBUG_DUMP=true` 시 request/inputs/response를 `logs/dumps/`에 저장. tools는 `available_tools.json`으로 1회만 저장하고 이후 `$tools_ref` 마커로 대체 |
| 구조화 로깅 | `logger.ts` | request 단위 추적 키(request_id) 연결, API key/AUTH_KEY/본문 마스킹, 파일 transport + 시작 시 백업 로테이션 |

설정은 전부 **env var 주입** (`config.ts`): upstream base URL, workflow auth, run mode, timeout, retry, log level/dir, debug dump 등. `{Home}/.env` 자동 로드.

---

## 6. 개발 지원 컴포넌트

### workflow-sandbox (`packages/workflow-sandbox`)
실제 AIU workflow 엔진 접속 없이 adapter를 개발/테스트하기 위한 가짜 upstream.
- **mock 모드**: deterministic 시나리오(ping-pong, tool_call_json, failure, timeout) 재현.
- **llm 모드**: `WORKFLOW_SANDBOX_LLM_MODE=llm` 시 실제 OpenAI-compatible API(기본 `gemini-3.1-flash-lite`)를 호출해, 실제 workflow LLM 노드와 동일한 System/User 프롬프트 구조로 응답 생성. `workflow-llm-system-prompt-spec.md` 전문을 PRE_CONTEXT로 사용.

### adapter-log (`packages/adapter-log`)
adapter stdout 파이프 또는 로그 파일을 tail해 브라우저에서 실시간 모니터링. Live Events / Sessions 탭, request_id/level/event 필터, `response_dumped` 이벤트의 메시지 타임라인 뷰어, 이벤트 방향 기호(▶/◀) 렌더링.

---

## 7. 진행 상태

HANDOFF.md / roadmap.md 기준:

| Phase | 목표 | 상태 |
| --- | --- | --- |
| Phase 1 | local ping-pong (adapter↔sandbox, blocking/streaming) | ✅ 완료 (ADP-001/004, SBX-001) |
| Phase 2 | opencode 계약 패리티 (tools/follow-up/title/usage/error) | ✅ 완료 (ADP-002/005, SBX-002) |
| Phase 3 | file/timeout/stop/retry/redaction 하드닝 | ✅ 완료 (ADP-003~051, SBX-003~029) |
| Phase 4 | 실제 workflow 그래프 구현 + 운영 | ✅ 완료 (WFL-001~003) |
| LOG | 로그 뷰어 도구 | ✅ 완료 (LOG-001~007) |

**미완료 항목:**
- `DOC-009 / DOC-010 / DOC-011` — opencode TUI 기반 통합 **E2E 휴먼 검증** (sandbox llm 모드 통합, 로그 뷰어 실시간 모니터링, `GET /v1/models` 호출 시점 검증).
- `ADP-049 Track A` — 확정된 rewrite 루프 패턴(`write→read→edit→edit→edit(no-change)→write→write`) 방지 규칙 3개를 **실제 AIU workflow LLM 노드 System 필드에 반영**.

---

## 8. 강점

- **계약 우선 + 역할 분리**가 일관되게 지켜져, 문서(`spec`)와 구현(`packages`)이 추적 가능하게 대응됨. roadmap의 ADP/SBX/WFL/DOC/LOG 태스크가 HANDOFF Work Log와 1:1로 연결되어 변경 이력이 매우 투명함.
- **개발 가능성(devability)이 잘 설계됨**: sandbox(mock/llm) + 로그 뷰어 + .http 스모크 + 덤프 기능으로 실제 workflow 없이도 전 경로를 재현/디버깅 가능.
- **실전에서 발견된 엣지 케이스가 풍부하게 반영됨**: 콘텐츠 필터 우회(태그 추출), 멀티턴 순서 보존, 비준수 JSON 복구, 루프 감지, Gemini 미지 필드 투과 등.
- 테스트가 spec 준수(`spec-compliance.test.ts`)와 task별로 촘촘함.

## 9. 리스크 / 한계

| 리스크 | 내용 |
| --- | --- |
| tool-call 신뢰성이 프롬프트에 종속 | native tool calling이 아니라 "LLM이 규칙대로 JSON을 뱉는다"는 가정에 의존. 모델/버전이 바뀌면 깨질 수 있음 (실제로 `gemini-2.5-flash`는 0 토큰 반환 버그로 모델 교체). 비준수 응답 복구 로직이 안전망이지만 완전하지 않음. |
| usage가 문자수 근사치 | `buildUsage`는 토큰이 아닌 문자수. opencode context bar 표시는 되지만 실제 토큰/비용과 단위가 다름 (`upstream_request_finished`의 `workflow_total_tokens`와 별개). |
| 사내 workflow 의존성 | adapter 계약은 안정적이나, 실제 tool 실행 성공 여부는 workflow LLM 노드 System 프롬프트 설정(`ADP-049 Track A`)에 달려 있고 이는 휴먼 작업으로 미완료. |
| E2E 검증 미완 | DOC-009/010/011가 미완이라, 통합 환경에서의 최종 동작은 런북 기준으로만 부분 확인됨. |
| 파일 입력 비지원 | OpenCode 프록시 경로는 파일 업로드 API를 쓰지 않고 text passthrough만 함 (설계상 non-goal). |

---

## 10. 학습/리뷰 우선순위

처음 보는 사람이 빠르게 이해하려면 다음 순서를 권장:

1. `docs/PRD.md` — 목적/범위/성공 기준
2. `docs/spec/opencode-interface-spec.md` — opencode ↔ adapter 계약 (요청/응답 예시 풍부)
3. `docs/spec/workflow-llm-system-prompt-spec.md` — ★ tool-call 에뮬레이션의 심장
4. `packages/adapter/src/workflow-client.ts` — `buildWorkflowInputs` / `classifyOutput` / `runStreaming`
5. `packages/adapter/src/adapter.ts` — 라우팅 + 루프 감지
6. `docs/task/roadmap.md` + `docs/HANDOFF.md` — 진행 상태와 변경 의도

---

## 11. 종합 판단

- 이 프로젝트는 "작은 변환 스크립트"가 아니라, **계약·구현·검증·도구·운영 절차가 분리된 소규모 제품형 통합 계층**이다.
- 핵심 기여는 **chat-only LLM 노드에 OpenAI tool-call 인터페이스를 입히는 변환 패턴**이며, 그 패턴은 "엄격한 JSON 출력 프롬프트 + adapter 측 견고한 파서/복구 로직"의 조합으로 구현되어 있다.
- 계약/문서 측면은 사실상 완성 단계이고, 남은 일은 대부분 **실제 workflow 노드 설정 반영(ADP-049 Track A)과 통합 E2E 휴먼 검증(DOC-009/010/011)** 이다. 이 둘이 닫히면 운영 전환 가능한 상태로 보인다.
