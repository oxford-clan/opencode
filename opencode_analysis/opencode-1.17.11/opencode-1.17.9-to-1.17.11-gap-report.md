# opencode 1.17.9 -> 1.17.11 Gap Analysis

> 분석 대상: `D:\AREA51\workspace\opencode`
> 현재 기준: `packages/opencode/package.json` version `1.17.11`, local HEAD `986846fbd`
> 비교 기준: `REPORT.md`에 등록된 `opencode 1.17.7` 분석 문서들과 `opencode 1.17.9` system prompt 갭 분석
> 작성 목적: 기존 분석 결과가 현재 최신 로컬 소스에서 여전히 유효한지, `aiu-opencode-adapter` 연동 관점에서 보완해야 할 갭이 무엇인지 정리한다.

---

## 1. 결론

현재 `opencode` 1.17.11 소스에서 `REPORT.md`의 핵심 결론은 대부분 유지된다.

특히 `aiu-opencode-adapter`가 일반 OpenAI-compatible custom provider로 붙는 경로에서는 다음 분석이 여전히 유효하다.

- provider request 준비 시 agent별 system prompt 교체 구조가 유지된다.
- `chat.headers` plugin hook에는 현재 agent 이름이 전달되지만, 기본 outbound provider header에는 agent 이름이 없다.
- `AGENTS.md`/`CLAUDE.md` 같은 프로젝트 지시문은 legacy 경로에서 system prompt의 `Instructions from:` 블록으로 들어간다.
- `/init`은 일반 command template이고, `/review`는 `subtask: true` command다.
- context usage UI는 assistant message의 token category 합계를 model context limit으로 나눠 표시한다.
- `task` permission deny는 일반 tool 실행이 아니라 subagent delegation 노출과 실행에 주로 영향을 준다.

다만 1.17.11 현재 소스에는 기존 1.17.7/1.17.9 분석이 충분히 다루지 못한 변화가 있다.

1. V2 Session Runner와 core `SystemContext` 경로가 더 구체화됐다.
2. legacy instruction loading과 V2/core instruction loading의 포함 범위가 다르다.
3. GitLab `GitLabWorkflowLanguageModel` 전용 `isWorkflow` 경로가 생겨 system prompt 전달 방식이 일반 OpenAI-compatible provider와 다르다.
4. overflow/auto compaction 판단에서 `tokens.total`이 있으면 우선 사용한다.
5. Task tool에는 background subagent, child permission derivation, doom-loop guard 등 운영 동작이 추가되어 기존 task permission 분석을 보강해야 한다.

따라서 `REPORT.md`의 기존 분석은 "legacy/OpenAI-compatible provider 경로 기준"으로는 계속 활용 가능하지만, 1.17.11 이후에는 V2/core runtime 및 workflow-provider 전용 경로와 구분해서 읽어야 한다.

---

## 2. 현재 소스 기준

확인 결과:

- 현재 checkout: `986846fbd`
- 현재 package version: `packages/opencode/package.json`의 `version` = `1.17.11`
- `REPORT.md`의 최신 직접 비교 문서: `opencode 1.17.7 -> 1.17.9 System Prompt 변경 분석`

주의:

- 로컬 git history에는 `REPORT.md`에 적힌 `v1.17.9@5c23e8841` ref가 존재하지 않아 `git diff 5c23e8841..HEAD` 방식의 직접 diff는 수행하지 못했다.
- 대신 현재 1.17.11 소스에서 기존 분석의 핵심 근거 지점을 직접 재검증했다.

---

## 3. 유지되는 분석

### 3.1 Legacy provider request의 system prompt 조립 구조

`packages/opencode/src/session/llm/request.ts`에서 system prompt는 여전히 다음 구조로 조립된다.

```ts
...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
...input.system,
...(input.user.system ? [input.user.system] : []),
```

의미:

- `explore`, `title`, `summary`, `compaction`처럼 agent prompt가 있으면 provider별 base prompt를 대체한다.
- `build`, `plan`, `general`처럼 agent prompt가 없으면 `SystemPrompt.provider(model)`가 모델 ID에 맞는 base prompt를 선택한다.
- 이후 environment, project instructions, skills, MCP instructions 등이 `input.system`으로 뒤에 붙는다.

이 결론은 1.17.7 분석 및 1.17.9 갭 분석과 동일하다.

근거 파일:

- `packages/opencode/src/session/llm/request.ts`
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/agent/agent.ts`

### 3.2 Agent 이름은 hook에는 있지만 기본 provider request에는 없다

`chat.headers` plugin hook에는 여전히 `agent: input.agent.name`이 전달된다.

```ts
const { headers } = yield* input.plugin.trigger(
  "chat.headers",
  {
    sessionID: input.sessionID,
    agent: input.agent.name,
    model: input.model,
    provider: input.provider,
    message: input.user,
  },
  { headers: {} },
)
```

하지만 일반 provider outbound header에는 agent 이름이 기본 포함되지 않는다.

```ts
{
  "x-session-affinity": input.sessionID,
  "X-Session-Id": input.sessionID,
  ...(input.parentSessionID ? { "x-parent-session-id": input.parentSessionID } : {}),
  "User-Agent": USER_AGENT,
}
```

따라서 adapter가 현재 agent 이름을 안정적으로 알아야 한다는 기존 결론도 유지된다.

권장 방향도 동일하다.

- opencode plugin 또는 core patch로 `x-opencode-agent` 같은 header를 추가한다.
- adapter가 그 값을 `opencode_agent` 또는 현재 adapter의 `agent` workflow input으로 넘긴다.
- system prompt나 model ID를 보고 agent를 추론하는 방식은 보조 수단으로만 취급한다.

근거 파일:

- `packages/opencode/src/session/llm/request.ts`

### 3.3 `/init`과 `/review` command 구조

현재 `packages/opencode/src/command/index.ts` 기준:

- `/init`은 `subtask` 설정이 없다. 즉 일반 prompt template을 현재 세션에 주입하는 경로다.
- `/review`는 `subtask: true`다.

core plugin command 정의에서도 같은 구조가 유지된다.

근거 파일:

- `packages/opencode/src/command/index.ts`
- `packages/core/src/plugin/command.ts`
- `packages/core/src/plugin/command/initialize.txt`
- `packages/core/src/plugin/command/review.txt`

adapter 관점:

- `/init` 자체는 user message로 전달되므로 adapter의 `messages` 전달 경로에서 빠지지 않는다.
- `/init` 중 모델이 `task` tool로 `explore` subagent를 호출하면 child session system prompt가 달라질 수 있고, 이때 adapter가 system prompt 일부만 추출하는 정책의 영향은 여전히 중요하다.

### 3.4 Context usage UI 계산 구조

UI context usage 계산은 여전히 assistant message의 token category 합계를 model limit으로 나눈다.

App:

- `packages/app/src/components/session/session-context-metrics.ts`
- `tokenTotal = input + output + reasoning + cache.read + cache.write`
- `usage = Math.round((total / limit) * 100)`

TUI:

- `packages/tui/src/component/prompt/index.tsx`
- `packages/tui/src/feature-plugins/sidebar/context.tsx`
- 최신 assistant message 선택 조건에 `tokens.output > 0` 경로가 남아 있다.

따라서 adapter가 `prompt_tokens`만 크게 주고 `completion_tokens`가 0에 가까운 경우, 일부 TUI 표면에서는 usage 선택이 기대와 다를 수 있다는 기존 분석은 유지된다.

---

## 4. 보완이 필요한 갭

### 4.1 V2 Session Runner와 SystemContext 경로

1.17.11 현재 `packages/core/src/session/runner/llm.ts`는 V2 runner의 provider turn을 상당 부분 구현하고 있다.

주요 동작:

- `SessionInput` pending steer/queue promotion
- `SessionContextEpoch.initialize/prepare`
- `SystemContextRegistry.load()`
- `SkillGuidance`와 `ReferenceGuidance` 결합
- `LLM.request(...)` 생성
- `llm.stream(request)` 1회 호출
- tool materialization 및 local tool settlement
- overflow compaction recovery

기존 1.17.7 분석은 legacy `SessionPrompt`와 `packages/opencode/src/session/llm/request.ts` 중심이다. 1.17.11에서는 V2 경로가 더 중요해졌으므로, 이후 분석 문서에서는 다음을 분리해야 한다.

| 구분 | 주요 파일 | adapter 영향 |
| --- | --- | --- |
| Legacy path | `packages/opencode/src/session/prompt.ts`, `packages/opencode/src/session/llm/request.ts` | 현재 OpenAI-compatible adapter 분석의 주 근거 |
| V2 path | `packages/core/src/session/runner/llm.ts`, `packages/core/src/system-context/*` | 향후 opencode가 V2를 기본화하면 system/context 전달 방식 검토 필요 |

근거 파일:

- `packages/core/src/session/runner/llm.ts`
- `packages/core/src/system-context/builtins.ts`
- `packages/core/src/instruction-context.ts`

### 4.2 Legacy instruction loading과 V2 instruction context 차이

legacy instruction service는 다음을 본다.

- global config `AGENTS.md`
- global `~/.claude/CLAUDE.md` unless disabled
- project-level `AGENTS.md`
- project-level `CLAUDE.md`
- deprecated `CONTEXT.md`
- config `instructions`에 지정된 local file, glob, URL
- read tool 결과에 붙는 nested instruction reminder

근거:

- `packages/opencode/src/session/instruction.ts`

반면 core V2 `InstructionContext`는 현재 다음 경로가 중심이다.

- global config `AGENTS.md`
- project upward `AGENTS.md`

근거:

- `packages/core/src/instruction-context.ts`

차이:

- legacy는 `CLAUDE.md`, `CONTEXT.md`, config `instructions` URL까지 다룬다.
- V2/core instruction context는 `AGENTS.md` 중심으로 단순하다.

adapter 관점:

- 현재 adapter의 `opencode_instructions` 추출이 legacy system prompt의 `Instructions from:` 블록에 의존한다면, legacy provider path에서는 여전히 맞다.
- V2 runner가 external provider 요청의 기본 경로가 될 경우, instruction baseline/update 이벤트가 어떤 형태로 provider request에 들어가는지 별도 분석이 필요하다.

### 4.3 GitLab workflow model 전용 `isWorkflow` 경로

1.17.11에는 `GitLabWorkflowLanguageModel` 전용 처리 경로가 있다.

`packages/opencode/src/session/llm.ts`:

- `const isWorkflow = language instanceof GitLabWorkflowLanguageModel`
- workflow model이면 `workflowModel.systemPrompt = prepared.system.join("\n")`
- workflow model이면 `workflowModel.toolExecutor`를 opencode tool system에 연결
- workflow model이면 `workflowModel.sessionPreapprovedTools`와 `approvalHandler`를 설정

`packages/opencode/src/session/llm/request.ts`:

```ts
const messages =
  isOpenaiOauth || input.isWorkflow
    ? input.messages
    : [
        ...system.map((x) => ({ role: "system", content: x })),
        ...input.messages,
      ]
```

의미:

- 일반 OpenAI-compatible provider에서는 system prompt가 `messages`의 `role:"system"`으로 들어간다.
- GitLab workflow model에서는 system prompt가 messages에 들어가지 않고 workflow model 객체의 `systemPrompt` 필드로 별도 전달된다.

adapter 관점:

- `aiu-opencode-adapter`가 `@ai-sdk/openai-compatible` provider로 등록되어 호출되는 한 기존 분석이 맞다.
- 그러나 opencode 내부의 "workflow model" 용어와 AIU Workflow adapter의 "workflow" 용어가 겹치므로 문서에서 반드시 구분해야 한다.
- 만약 향후 AIU adapter가 opencode 내부 workflow provider 형태로 붙는다면, system prompt 추출 방식은 완전히 재검토해야 한다.

근거 파일:

- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/request.ts`
- `packages/core/src/plugin/provider/gitlab.ts`

### 4.4 Overflow/auto compaction에서 `tokens.total` 우선 사용

기존 context usage 분석은 UI가 persisted token categories를 합산하며 `totalTokens`를 직접 사용하지 않는다는 점을 설명했다. 이 설명은 UI 관점에서는 여전히 맞다.

하지만 auto compaction overflow 판단에서는 현재 `tokens.total`이 있으면 우선 사용한다.

```ts
const count =
  input.tokens.total || input.tokens.input + input.tokens.output + input.tokens.cache.read + input.tokens.cache.write
return count >= usable(input)
```

의미:

- provider usage의 `totalTokens`가 assistant message token object에 남아 있으면 overflow 판단에 영향을 준다.
- adapter가 `total_tokens`를 부정확하게 크게 주면 자동 compaction이 예상보다 빨리 발생할 수 있다.
- adapter가 `total_tokens`를 0 또는 누락시키고 input/output만 주면 category 합산 경로가 사용된다.

근거 파일:

- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/overflow.ts`

adapter 권장:

- `prompt_tokens`, `completion_tokens`, `total_tokens` 사이의 일관성을 유지해야 한다.
- 실제 tokenizer가 없다면 character count를 쓰더라도 `model.limit.context`와 compaction 설정이 같은 단위의 proxy처럼 해석된다는 점을 운영 문서에 명시해야 한다.

### 4.5 Task tool 운영 동작 보강 필요

기존 `task` permission deny 분석은 핵심적으로 맞다. 다만 현재 `TaskTool`에는 다음 추가 동작이 있다.

- `background: true` 지원
- `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` 필요
- `task_id`를 통한 기존 subagent session resume
- parent session permission과 subagent permission을 결합하는 `deriveSubagentSessionPermission`
- subagent에 `todowrite`/`task` 기본 deny를 보강
- `experimental.primary_tools`를 child session에서 deny
- foreground task도 background job infrastructure를 통해 실행

또한 `SessionProcessor`에는 같은 tool/input 반복을 감지해 `doom_loop` permission을 묻는 guard가 있다.

adapter 관점:

- adapter가 tool loop 반복 방지를 자체적으로 수행하더라도, opencode 쪽에도 doom-loop permission guard가 있다.
- adapter가 의미 오류를 과도하게 차단하기보다 opencode의 `invalid` tool / repair path / permission path에 맡기는 설계는 여전히 타당하다.
- 다만 workflow LLM이 `task` tool을 자주 호출하는 경우 background/subagent session 증가와 approval UX까지 고려해야 한다.

근거 파일:

- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/agent/subagent-permissions.ts`
- `packages/opencode/src/session/processor.ts`

---

## 5. Adapter 연동 관점 요약

### 5.1 기존 adapter 설계가 계속 맞는 부분

- 일반 OpenAI-compatible custom provider로 붙는다면 system prompt는 계속 system message로 관찰 가능하다.
- `<env>`, `<available_skills>`, `Instructions from:` 추출 전략은 legacy provider path에서 계속 동작한다.
- agent 이름은 기본 request body/header에 없으므로 명시 전달이 필요하다는 결론은 유지된다.
- tool calls는 Chat Completions 응답의 `tool_calls`로 돌려줘야 한다는 계약도 유지된다.
- usage/context 표시 문제는 여전히 provider usage object와 model metadata에 의존한다.

### 5.2 새로 주의해야 할 부분

- `isWorkflow`는 GitLab workflow model 전용 경로다. AIU adapter의 workflow와 혼동하면 안 된다.
- V2 runner가 활성화 또는 기본화될 경우, system/context 전달 경로는 legacy 분석만으로 충분하지 않다.
- V2/core `InstructionContext`는 legacy `Instruction`과 포함 범위가 다르다. 특히 `CLAUDE.md`와 config `instructions` 처리 여부를 별도로 확인해야 한다.
- `totalTokens`는 UI percentage에는 직접 쓰이지 않더라도 overflow/compaction에는 영향을 줄 수 있다.
- task/subagent 기능은 background mode와 child permission derivation까지 포함해 더 복잡해졌다.

---

## 6. REPORT.md 반영 권장

`REPORT.md`에는 `opencode 1.17.11` 섹션을 추가하고, 이 문서를 다음 목적의 갭 분석으로 등록하는 것이 좋다.

- 1.17.9 이후 현재 1.17.11 소스 기준으로 기존 분석의 유지/수정 지점을 검증
- legacy/OpenAI-compatible provider 분석과 V2/core runner 분석의 경계 명확화
- AIU adapter가 계속 의존해도 되는 계약과 재검토해야 할 경로 구분

---

## 7. 후속 분석 과제

1. V2 runner가 실제 CLI/TUI 기본 실행 경로에서 언제 사용되는지 확인한다.
2. V2 `SystemContext` baseline/update가 external provider request에 어떤 message shape로 들어가는지 end-to-end로 추적한다.
3. `CLAUDE.md`가 V2/core path에서 의도적으로 제외된 것인지, legacy 호환 갭인지 확인한다.
4. `isWorkflow` GitLab path와 AIU adapter path를 비교하는 용어 정리 문서를 추가한다.
5. adapter usage 정책에서 `total_tokens`가 compaction에 주는 영향을 별도 운영 가이드로 정리한다.

---

## 8. 확인한 주요 파일

- `packages/opencode/package.json`
- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/request.ts`
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/session/instruction.ts`
- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/overflow.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/prompt.ts`
- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/tool/task.ts`
- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/permission/index.ts`
- `packages/opencode/src/command/index.ts`
- `packages/core/src/session/runner/llm.ts`
- `packages/core/src/session/runner/max-steps.ts`
- `packages/core/src/system-context/builtins.ts`
- `packages/core/src/instruction-context.ts`
- `packages/core/src/plugin/agent.ts`
- `packages/core/src/plugin/command.ts`
- `packages/core/src/plugin/provider/gitlab.ts`
- `packages/app/src/components/session/session-context-metrics.ts`
- `packages/tui/src/component/prompt/index.tsx`
- `packages/tui/src/feature-plugins/sidebar/context.tsx`
