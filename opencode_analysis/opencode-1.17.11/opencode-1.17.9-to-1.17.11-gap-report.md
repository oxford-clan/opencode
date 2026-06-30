# opencode 1.17.9 -> 1.17.11 분석 GAP 보고서

> 분석 대상: `D:\AREA51\workspace\opencode`
> 현재 소스 기준: `packages/opencode/package.json` `1.17.11`, `HEAD` = `986846fbd-dirty`
> 기존 분석 기준: `REPORT.md`의 `opencode 1.17.7` 분석 묶음 및 `opencode 1.17.9` system prompt diff 보고서
> 작성 목적: 기존 분석 내용과 현재 소스 사이에서 `aiu-opencode-adapter` 연동 판단에 영향을 줄 수 있는 차이를 정리한다.

---

## 0. 결론

현재 체크아웃된 소스는 `opencode` 1.17.11이며, 기존 `REPORT.md`가 다루는 최신 분석 기준은 1.17.9다.

1.17.9 분석의 핵심 결론인 **legacy V1 provider request에서 agent prompt가 있으면 모델별 base prompt를 대체한다**는 구조는 1.17.11에서도 유지된다.

다만 1.17.11 현재 소스에는 기존 분석에 없거나 보강이 필요한 차이가 있다.

1. System context에 `<available_references>`와 `<mcp_instructions>` 블록이 추가될 수 있다. 현재 adapter가 `<env>`, `Instructions from:`, `<available_skills>`만 추출한다면 이 정보는 workflow로 전달되지 않을 수 있다.
2. `GitLabWorkflowLanguageModel` 전용 경로가 생겼다. 이 경로는 system prompt를 Chat Completions `messages`에 넣지 않고 `workflowModel.systemPrompt`로 별도 전달하며, tool execution도 provider-side workflow 모델에 bridge한다. AIU adapter의 OpenAI-compatible provider 경로와는 별개의 내부 workflow provider 경로로 구분해야 한다.
3. native LLM runtime이 opt-in으로 추가됐다. OpenAI/OpenAI-compatible/Anthropic 일부 provider는 `OPENCODE_EXPERIMENTAL_NATIVE_LLM` 활성화 시 AI SDK 대신 `@opencode-ai/llm` native runtime을 탈 수 있다.
4. V2 session runner의 compaction이 1.17.9 분석보다 더 구체화됐다. `compactIfNeeded`, overflow recovery, `<conversation-checkpoint>` message가 현재 core runner 흐름에 들어 있다.
5. structured output 요청은 `StructuredOutput` tool과 추가 system prompt를 주입한다. adapter가 JSON schema format 요청을 받는 경우 일반 tool loop와 다른 `toolChoice: "required"` 흐름을 고려해야 한다.
6. usage 정규화의 큰 방향은 유지된다. AI SDK usage에서 `inputTokens`, `outputTokens`, reasoning/cache token 필드를 뽑아 LLM event로 넘기는 구조는 현재도 확인된다.

---

## 1. 기준 버전 확인

현재 소스 기준:

- `packages/opencode/package.json:3` — `"version": "1.17.11"`
- `package.json:7` — `"packageManager": "bun@1.3.14"`
- `package.json:65` — root catalog의 `ai` 버전은 `6.0.168`
- `packages/opencode/package.json:122` — `gitlab-ai-provider` `6.9.3`

로컬 git에는 기존 1.17.9 분석 보고서에 적힌 `v1.17.9` tag 또는 `5c23e8841` object가 없었다. 따라서 이 보고서는 git diff가 아니라 다음 방식으로 작성했다.

- 기존 1.17.9 gap 보고서의 결론을 기준선으로 사용.
- 현재 1.17.11 소스 파일을 직접 열람해 핵심 계약 유지/변경 여부를 검증.

현재 worktree의 `dirty` 상태는 분석 지침 문서인 `AGENTS.md` 변경 때문이다.

---

## 2. Legacy V1 system prompt 구조

### 유지되는 부분

`packages/opencode/src/session/llm/request.ts`의 system prompt 조립 구조는 기존 분석과 같은 계열이다.

- `packages/opencode/src/session/llm/request.ts:58` — `const system = [...]`
- `packages/opencode/src/session/llm/request.ts:57` — OpenAI OAuth 여부 판정
- `packages/opencode/src/session/llm/request.ts:70` — `experimental.chat.system.transform` hook
- `packages/opencode/src/session/llm/request.ts:101` — `isOpenaiOauth || input.isWorkflow`일 때 messages 처리 분기

핵심 의미:

- `input.agent.prompt`가 있으면 모델별 provider base prompt 대신 agent prompt가 첫 system block이 된다.
- agent prompt가 없으면 `SystemPrompt.provider(input.model)`가 모델 ID에 맞는 base prompt를 고른다.
- 그 뒤에 environment, instruction, MCP, skills 등 session-level system context가 붙는다.

### 보강해야 할 부분

1.17.11에는 `experimental.chat.system.transform` 이후 system array가 2개 이상으로 나뉠 수 있다. 첫 header가 유지되고 system part가 3개 이상이면 나머지를 두 번째 system part로 합친다.

adapter 관점:

- adapter가 모든 `role:"system"` 메시지를 합쳐 처리한다면 큰 문제는 없다.
- system message가 하나라고 가정하는 분석이나 구현은 더 이상 안전하지 않다.

---

## 3. System context 신규/누락 가능 블록

기존 adapter 중심 분석은 다음 세 블록을 주요 전달 대상으로 봤다.

- `<env>`
- `Instructions from: ...`
- `<available_skills>`

1.17.11 현재는 여기에 더해 다음 블록이 생길 수 있다.

### 3.1 Project references

`packages/opencode/src/session/system.ts`에서 reference guidance를 environment system context에 포함한다.

- `packages/opencode/src/session/system.ts:79` — `<available_references>`
- `packages/opencode/src/session/system.ts:91` — `</available_references>`

V2 쪽에도 같은 개념이 있다.

- `packages/core/src/reference/guidance.ts` — reference list를 `<available_references>`로 렌더링

adapter 영향:

- 현재 adapter가 `<env>`만 추출하면 reference 경로와 설명은 누락된다.
- 사내 workflow가 프로젝트 reference 기능을 써야 한다면 `opencode_references` 같은 별도 workflow input 또는 system context passthrough가 필요하다.

### 3.2 MCP instructions

`packages/opencode/src/session/system.ts`는 MCP server가 제공한 instructions를 system context로 추가할 수 있다.

- `packages/opencode/src/session/system.ts:110` — `mcp(...)`
- `packages/opencode/src/session/system.ts:118` — `<mcp_instructions>`
- `packages/opencode/src/session/system.ts:124` — `</mcp_instructions>`

adapter 영향:

- MCP instructions는 tool schema와 별개로 모델이 따라야 할 server-level 지시다.
- adapter가 이를 workflow로 넘기지 않으면, MCP tool은 노출되더라도 server별 사용 지시가 누락될 수 있다.

### 3.3 Skills

`<available_skills>`는 유지된다.

- `packages/opencode/src/session/system.ts:102` — skills 안내 문구
- `packages/opencode/src/skill/index.ts:330` — `Skill.fmt(...)`
- `packages/opencode/src/skill/index.ts:335` — `<available_skills>`
- `packages/core/src/skill/guidance.ts` — V2 skill guidance도 같은 tag를 사용

adapter 영향:

- 기존 `opencode_available_skills` 추출 전략은 여전히 유효하다.
- 다만 V2/core guidance는 skill location을 포함하지 않는 반면 V1 `Skill.fmt(..., { verbose: true })`는 `<location>`을 포함한다. workflow가 skill loading을 에뮬레이션하거나 설명만 쓰는지에 따라 차이가 있다.

---

## 4. Agent prompt 및 agent 정의

### Legacy agent

`packages/opencode/src/agent/agent.ts`의 legacy agent 구조는 기존 분석과 같은 큰 틀을 유지한다.

- `packages/opencode/src/agent/agent.ts:140` — built-in `agents`
- `packages/opencode/src/agent/agent.ts:141` — `build`
- `packages/opencode/src/agent/agent.ts:156` — `plan`
- `packages/opencode/src/agent/agent.ts:182` — `general`
- `packages/opencode/src/agent/agent.ts:196` — `explore`
- `packages/opencode/src/agent/agent.ts:219` — `compaction`
- `packages/opencode/src/agent/agent.ts:234` — `title`
- `packages/opencode/src/agent/agent.ts:250` — `summary`

`explore`, `compaction`, `title`, `summary`는 여전히 prompt 파일을 가진다.

- `packages/opencode/src/agent/agent.ts:214` — `PROMPT_EXPLORE`
- `packages/opencode/src/agent/agent.ts:224` — `PROMPT_COMPACTION`
- `packages/opencode/src/agent/agent.ts:248` — `PROMPT_TITLE`
- `packages/opencode/src/agent/agent.ts:263` — `PROMPT_SUMMARY`

### V2 agent plugin

V2 plugin 쪽에서는 default agent에도 explicit system이 들어간다.

- `packages/core/src/plugin/agent.ts:12` — `BUILD_SYSTEM`
- `packages/core/src/plugin/agent.ts:125` — default agent update
- `packages/core/src/plugin/agent.ts:127` — `item.system ??= BUILD_SYSTEM`
- `packages/core/src/plugin/agent.ts:163` — `explore`
- `packages/core/src/plugin/agent.ts:184` — `compaction`
- `packages/core/src/plugin/agent.ts:191` — `title`
- `packages/core/src/plugin/agent.ts:198` — `summary`

adapter 영향:

- 기존 1.17.9 보고서는 "V2에서도 agent별 system이 있으면 baseline 앞에 온다"고 정리했다. 1.17.11에서는 default/build 계열에도 V2 system이 더 명확하게 존재한다.
- V1 adapter 경로에서는 `build`가 여전히 모델별 base prompt를 쓰지만, V2 core runner 분석에는 default agent system을 별도로 반영해야 한다.

---

## 5. GitLab Workflow provider 경로 추가

1.17.11의 가장 큰 구조적 gap은 `GitLabWorkflowLanguageModel` 특수 처리다.

근거:

- `packages/opencode/src/session/llm.ts:13` — `GitLabWorkflowLanguageModel` import
- `packages/opencode/src/session/llm.ts:105` — `isWorkflow`
- `packages/opencode/src/session/llm.ts:119` — workflow model branch
- `packages/opencode/src/session/llm.ts:127` — `workflowModel.toolExecutor`
- `packages/opencode/src/session/llm.ts:150` — `sessionPreapprovedTools`
- `packages/opencode/src/session/llm.ts:156` — `approvalHandler`
- `packages/opencode/src/session/llm/request.ts:101` — `isWorkflow`이면 system messages를 일반 messages에 넣지 않음

provider 구성:

- `packages/opencode/src/provider/provider.ts:130` — `gitlab-ai-provider` bundled provider
- `packages/opencode/src/provider/provider.ts:591` 이후 — GitLab provider custom loader
- `packages/opencode/src/provider/provider.ts:628` 이후 — `duo-workflow-*` 모델 처리

의미:

- 이 경로는 AIU adapter처럼 외부 OpenAI-compatible HTTP adapter에 tool call JSON을 맡기는 구조가 아니다.
- opencode가 workflow language model 객체에 직접 `systemPrompt`, `toolExecutor`, approval handler를 꽂는다.
- workflow provider가 tool call을 요청하면 opencode 내부 tool system이 실행하고 결과를 다시 workflow provider에 돌려준다.

adapter 영향:

- AIU adapter 분석에서는 이 경로를 "참고 가능한 내부 workflow 통합 사례"로 볼 수 있다.
- 하지만 AIU adapter가 받는 OpenAI-compatible request body와는 wire contract가 다르므로, 이 경로의 system/tool 전달 방식을 그대로 adapter 계약으로 가정하면 안 된다.

---

## 6. Native LLM runtime opt-in

1.17.11에는 AI SDK 외에 native runtime 선택지가 있다.

근거:

- `packages/opencode/src/effect/runtime-flags.ts` — `OPENCODE_EXPERIMENTAL_NATIVE_LLM`
- `packages/opencode/src/session/llm.ts:226` — native runtime 분기
- `packages/opencode/src/session/llm/native-runtime.ts` — native runtime implementation

지원 범위:

- `packages/opencode/src/session/llm/native-runtime.ts`는 providerID가 `openai`, `anthropic`, 또는 `opencode*`인 경우를 우선 지원한다.
- npm package는 `@ai-sdk/openai`, `@ai-sdk/openai-compatible`, `@ai-sdk/anthropic` 중심이다.

adapter 영향:

- AIU adapter가 `@ai-sdk/openai-compatible` provider로 등록되어 있고 `OPENCODE_EXPERIMENTAL_NATIVE_LLM`이 켜지면, 기존 AI SDK wire behavior와 다른 native request path를 탈 가능성이 있다.
- native runtime은 `@opencode-ai/llm` request로 낮춘 뒤 transport한다. OpenAI-compatible adapter가 AI SDK의 exact Chat Completions request shape에 의존한다면 E2E 확인이 필요하다.
- 기본값은 opt-in이므로, 플래그가 꺼져 있으면 기존 AI SDK path 분석이 우선이다.

---

## 7. Tool call repair 및 schema strict 처리

현재 AI SDK path에는 invalid tool call repair가 들어 있다.

- `packages/opencode/src/session/llm.ts:296` — `experimental_repairToolCall`
- `packages/opencode/src/session/llm.ts:317` — `activeTools`에서 `invalid` 제외

동작:

- tool name 대소문자만 다른 경우 lowercase tool로 repair한다.
- repair할 수 없으면 `invalid` tool에 `{ tool, error }` JSON을 넣는다.

또한 OpenAI Responses 계열 provider에 대해 tool schema strict false를 강제로 넣는다.

- `packages/opencode/src/session/llm/request.ts:149` — strict false 설명
- `packages/opencode/src/session/llm/request.ts:157` — `tools[key] = { ...tools[key], strict: false }`

adapter 영향:

- adapter가 workflow output에서 unknown tool이나 malformed arguments를 과도하게 차단하면 opencode의 repair/invalid tool 흐름을 방해할 수 있다.
- 기존 adapter 분석의 "tool call envelope guard는 의미 오류를 OpenCode에 위임"이라는 방향은 1.17.11에서도 타당하다.

---

## 8. Structured output path

1.17.11 legacy prompt loop는 JSON schema structured output 요청에 대해 별도 tool과 system prompt를 추가한다.

근거:

- `packages/opencode/src/session/prompt.ts:74` — `STRUCTURED_OUTPUT_DESCRIPTION`
- `packages/opencode/src/session/prompt.ts:82` — `STRUCTURED_OUTPUT_SYSTEM_PROMPT`
- `packages/opencode/src/session/prompt.ts:1243` — `StructuredOutput` tool 추가
- `packages/opencode/src/session/prompt.ts:1270` — structured output system prompt 추가
- `packages/opencode/src/session/prompt.ts:1284` — `toolChoice: "required"`

adapter 영향:

- 일반 코딩 agent tool loop와 달리, structured output 요청은 모델이 반드시 `StructuredOutput` tool을 호출해야 한다.
- workflow JSON output 에뮬레이션과 충돌할 수 있다. workflow system prompt가 자체 `{"answer","tool_calls"}` envelope를 강제하는 경우, opencode가 요구하는 `StructuredOutput` tool call을 어떻게 표현할지 별도 검토가 필요하다.

---

## 9. V2 compaction gap

기존 1.17.9 보고서는 V2 runner의 step-limit 변화를 언급했지만, 1.17.11 현재 V2 compaction 흐름은 더 구체화되어 있다.

근거:

- `packages/core/src/session/runner/llm.ts:108` — `SessionCompaction.make(...)`
- `packages/core/src/session/runner/llm.ts:210` — `compactIfNeeded(...)`
- `packages/core/src/session/runner/llm.ts:365` — overflow recovery 가능한 `runTurn`
- `packages/core/src/session/compaction.ts:16` — summary template
- `packages/core/src/session/compaction.ts:166` — `buildPrompt(...)`
- `packages/core/src/session/compaction.ts:177` — `compactAfterOverflow(...)`
- `packages/core/src/session/compaction.ts:230` — `compactIfNeeded(...)`
- `packages/core/src/session/runner/to-llm-message.ts:152` — `<conversation-checkpoint>`

adapter 영향

- V2 path에서는 compaction summary가 `compaction` message로 history에 재주입되고, provider-facing message에는 `<conversation-checkpoint>`가 들어갈 수 있다.
- adapter가 V2 request를 받는 경우, 이 synthetic checkpoint를 일반 user prompt처럼 workflow에 전달하게 된다.
- 기존 V1 `/compact` marker 중심 분석만으로는 V2 compaction request/history shape를 설명하기 부족하다

---

## 10. Usage/context 계산

usage normalization의 핵심은 현재도 유지된다.

근거:

- `packages/opencode/src/session/llm/ai-sdk.ts:44` — AI SDK usage extraction
- `packages/opencode/src/session/llm/ai-sdk.ts:56` — `inputTokens`
- `packages/opencode/src/session/llm/ai-sdk.ts:57` — `outputTokens`
- `packages/opencode/src/session/llm/ai-sdk.ts:60` — cache read tokens
- `packages/opencode/src/session/llm/ai-sdk.ts:87` — `finish-step`
- `packages/opencode/src/session/llm/ai-sdk.ts:116` — `finish` total usage

기존 분석 유지:

- adapter가 문자 수를 `prompt_tokens`/`completion_tokens`로 반환하면 opencode는 이를 token으로 받아들인다.
- context percentage는 모델 metadata의 context limit과 저장된 token category 합산에 의존한다.

보강점:

- AI SDK 6 path에서는 `finish-step`과 `finish` 이벤트가 모두 usage를 가질 수 있다.
- provider metadata가 reasoning/tool/result metadata와 함께 더 많이 보존된다. adapter가 provider-specific metadata를 추가로 흘려보낼 경우 opencode가 일부를 보존할 수 있다.

---

## 11. 기존 분석 문서별 유효성 평가

| 기존 문서 | 1.17.11 평가 | 보강 필요 |
| --- | --- | --- |
| System Prompt 관리 방식 분석 | 대체로 유효 | `<available_references>`, `<mcp_instructions>`, multi system part, GitLab workflow path 추가 필요 |
| `/init` 및 Agent별 System Prompt 영향 검토 | 대체로 유효 | V2 default agent system, references/MCP 누락 리스크 추가 필요 |
| agent 정보 식별 가능성 검토 | 유효 | GitLab workflow provider는 별도 내부 경로라는 예외 설명 필요 |
| Session 관리 정책 분석 | 부분 유효 | V2 durable runner/compaction 흐름은 별도 최신 분석 필요 |
| Context Usage 계산 방식 분석 | 대체로 유효 | AI SDK 6 event field 확인 및 native runtime 사용 시 재검증 필요 |
| Context 자동 압축 분석 | V1은 유효, V2는 보강 필요 | `SessionCompaction.make`, `<conversation-checkpoint>`, overflow compaction 추가 |
| task 권한 deny 영향 분석 | 대체로 유효 | V2 PermissionV2 ruleset과 workflow tool approval 경로는 별도 확인 필요 |
| Custom Provider Spec/Guide | 대체로 유효 | native runtime opt-in과 GitLab workflow custom provider 사례 추가 필요 |

---

## 12. adapter 관점 권장 후속 분석

1. `opencode_references` 전달 필요성 검토
   - `<available_references>`가 adapter workflow에 필요한지 결정한다.
   - 필요하면 `opencode_references` optional input 또는 generic `opencode_system_context_extra` input을 설계한다.

2. `opencode_mcp_instructions` 전달 필요성 검토
   - MCP server instructions를 누락해도 되는지, tool schema만으로 충분한지 확인한다.

3. native runtime OFF 고정 여부 결정
   - AIU adapter 운영에서 `OPENCODE_EXPERIMENTAL_NATIVE_LLM`을 금지할지, native path에서도 OpenAI-compatible adapter request가 호환되는지 E2E 검증할지 정한다.

4. structured output과 workflow JSON envelope 충돌 검토
   - `format.type === "json_schema"` 요청이 AIU adapter 경로로 들어올 때 workflow prompt가 `StructuredOutput` tool call을 만들 수 있는지 검증한다.

5. V2 session runner 별도 분석
   - 현재 REPORT의 많은 문서는 legacy V1 session prompt loop를 중심으로 한다.
   - V2가 실제 사용 경로로 전환될 경우 provider request, system context, compaction, tool settlement를 별도 최신 문서로 정리해야 한다.

---

## 13. 최종 판단

1.17.11 현재 소스는 기존 1.17.9 분석과 완전히 다른 구조로 바뀐 것은 아니다. AIU adapter가 주로 의존하는 legacy OpenAI-compatible provider request 경로는 큰 틀에서 유지된다.

그러나 분석 gap은 분명하다.

- system context가 `<env>`/instructions/skills보다 넓어졌다.
- workflow라는 이름의 내부 GitLab provider 경로가 생겨 adapter의 외부 workflow bridge와 개념적으로 혼동될 수 있다.
- native runtime과 V2 runner가 강화되면서 "AI SDK Chat Completions request만 보면 된다"는 분석 범위가 장기적으로는 부족해졌다.

따라서 현재 adapter 연동 안정성 판단에는 기존 분석을 계속 사용할 수 있지만, `references`, `mcp_instructions`, `structured output`, `native runtime`, `V2 compaction`은 별도 보강 분석 대상으로 등록하는 것이 안전하다.
