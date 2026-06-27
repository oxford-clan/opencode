# aiu-opencode-adapter의 opencode agent 정보 식별 가능성 검토

> 검토 대상: `D:\AREA51\workspace\opencode` 및 `D:\AREA51\workspace\aiu-opencode-adapter`
> 작성 일자: 2026-06-19
> 목적: opencode가 adapter로 보내는 OpenAI Chat Completions API 요청만으로 현재 실행 중인 agent 정보를 알 수 있는지, 그리고 명시 전달이 필요하면 어디에 붙이는 것이 안전한지 정리한다.

---

## 1. 결론

현재 `aiu-opencode-adapter`는 opencode의 API 요청만 보고 "현재 agent가 build인지, plan인지, explore인지"를 명시적으로 알 수 없다.

adapter가 받는 OpenAI 호환 요청 본문에는 `model`, `messages`, `tools`, `tool_choice`, `stream` 등만 있고 `agent` 필드는 없다. adapter가 읽는 header도 현재는 `x-request-id`, `x-session-affinity`, `x-session-id`, `user-agent` 중심이며 agent 이름을 담는 header는 없다.

다만 opencode 내부에는 provider 요청을 만들 때 이미 `input.agent.name`이 존재한다. 특히 `packages/opencode/src/session/llm/request.ts`의 `chat.headers` plugin hook에는 `{ agent: input.agent.name }`이 전달된다. 따라서 opencode plugin 또는 core patch로 `x-opencode-agent` 같은 header를 추가하면 adapter가 안정적으로 알 수 있다.

간접 추론은 가능하지만 권장하지 않는다. agent별 system prompt, tools 목록, model override 조합으로 추정할 수는 있으나 custom agent, permission 설정, model 설정, system prompt 전달 정책에 따라 쉽게 깨진다.

---

## 2. adapter 요청 계약에서 agent 필드가 없음

`aiu-opencode-adapter`의 OpenAI 호환 request 타입은 `packages/adapter/src/workflow-client.ts:24-31` 기준 다음 범위다.

```ts
export interface ChatRequest {
  model?: string;
  messages?: ChatMessage[];
  tools?: Array<{ type: string; function: { name: string; [key: string]: unknown } }>;
  tool_choice?: string | Record<string, unknown>;
  max_tokens?: number;
  stream?: boolean;
  files?: unknown[];
}
```

여기에 `agent`, `opencode_agent`, `mode` 같은 필드는 없다.

workflow 입력 계약도 `packages/adapter/src/workflow-client.ts:40-49` 기준 다음 범위다.

```ts
export interface WorkflowInputs {
  request_id: string;
  model: string;
  opencode_env: string;
  opencode_available_skills: string;
  opencode_instructions?: string;
  messages: string;
  messages_2: string;
  tools?: string;
  tool_choice?: string;
}
```

현재 workflow로 전달되는 것은 model, env, skills, project instructions, non-system message history, tools, tool choice다. agent 이름을 담는 입력은 없다.

---

## 3. adapter 추출 로직도 agent를 만들지 않음

`buildWorkflowInputs()`는 `packages/adapter/src/workflow-client.ts:411-442`에서 system message와 history message를 분리한 뒤, system prompt에서 다음 정보만 추출한다.

- `<env>` -> `opencode_env`
- `<available_skills>` -> `opencode_available_skills`
- `Instructions from:` 블록 -> `opencode_instructions`

그 뒤 system이 아닌 메시지를 JSON 문자열로 직렬화해 `messages`와 `messages_2`에 넣고, tools와 tool choice가 있으면 별도 문자열로 넘긴다.

이 경로에는 agent 이름을 파싱하거나 생성하는 단계가 없다. 즉 adapter 입장에서는 "요청이 어떤 agent에서 왔다"는 정보가 현재 데이터 모델에 없다.

---

## 4. adapter HTTP handler도 agent header를 읽지 않음

`packages/adapter/src/adapter.ts`에서 request metadata는 `adapterRequestId`, `sessionId`, `userId`만 가진다 (`adapter.ts:153-156`).

```ts
const requestMeta = {
  adapterRequestId: context.adapterRequestId,
  sessionId: context.sessionId,
  userId: context.userId,
};
```

session 식별자는 `adapter.ts:249-254`에서 다음 header로 만든다.

```ts
const adapterRequestId =
  (request.headers["x-request-id"] as string | undefined) ?? randomUUID();
const sessionId =
  (request.headers["x-session-affinity"] as string | undefined) ??
  (request.headers["x-session-id"] as string | undefined) ??
  adapterRequestId;
```

로그의 `stage: "agent"`는 실제 agent 이름이 아니라 처리 단계명이다 (`adapter.ts:160-162`, `adapter.ts:231-232`). 이 값을 `build`, `plan`, `explore` 같은 agent 식별자로 해석하면 안 된다.

---

## 5. opencode 내부에는 agent 정보가 있음

opencode provider 요청 준비 단계에서는 현재 agent가 분명히 존재한다.

첫째, system prompt 조립 시 agent 자체 prompt가 있으면 base system prompt를 agent prompt로 교체한다 (`packages/opencode/src/session/llm/request.ts:58-66`).

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
]
```

둘째, `chat.headers` plugin hook에 agent 이름을 넘긴다 (`packages/opencode/src/session/llm/request.ts:135-146`).

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
  {
    headers: {},
  },
)
```

즉 opencode 안에서는 "현재 provider turn의 agent 이름"을 알고 있으며, 외부 provider request에 header로 추가할 확장 지점도 이미 있다.

---

## 6. 기본 provider header에는 agent가 없음

opencode가 일반 provider로 보내는 기본 header는 `packages/opencode/src/session/llm/request.ts:176-190` 기준 session 중심이다.

```ts
{
  "x-session-affinity": input.sessionID,
  "X-Session-Id": input.sessionID,
  ...(input.parentSessionID ? { "x-parent-session-id": input.parentSessionID } : {}),
  "User-Agent": USER_AGENT,
}
```

`...headers`로 plugin이 만든 header가 마지막에 병합되므로, plugin이 `x-opencode-agent`를 넣으면 adapter가 받을 수 있다. 하지만 기본 상태에서는 agent header가 없다.

---

## 7. 현재 요청에서 확실히 알 수 있는 정보와 알 수 없는 정보

현재 adapter가 안정적으로 알 수 있는 정보:

| 정보 | 경로 | 비고 |
| --- | --- | --- |
| session ID | `x-session-affinity` 또는 `x-session-id` | provider turn/session 묶음 식별 가능 |
| request ID | `x-request-id` 또는 adapter 생성 UUID | adapter 내부 추적용 |
| model ID | request body `model` | per-agent model override가 있어도 agent 이름 자체는 아님 |
| tools 목록 | request body `tools` | permission 결과 추정은 가능하지만 결정적이지 않음 |
| env/skills/instructions | system prompt 추출 | 현재 adapter 계약에 포함 |
| parent session | `x-parent-session-id` | child session 여부 추정 가능, agent 이름은 아님 |

현재 adapter가 명시적으로 알 수 없는 정보:

| 정보 | 이유 |
| --- | --- |
| 현재 agent 이름 | body/header/workflow input 어디에도 없음 |
| agent mode | `primary`, `subagent`, `all` 같은 mode가 전달되지 않음 |
| agent-specific prompt 원문 | adapter는 system prompt 전체를 전달하지 않고 일부 블록만 추출 |
| agent permission 원본 | 최종 tools 목록은 보이나 agent 설정 자체는 없음 |

---

## 8. 간접 추론이 취약한 이유

agent-specific system prompt로 일부 추정은 가능하다. 예를 들어 `explore`는 `You are a file search specialist...`로 시작하는 별도 prompt를 쓴다. `title`, `summary`, `compaction`도 별도 prompt가 있다.

하지만 이 방식은 adapter 구조와 운영 설정에서 안정적이지 않다.

- adapter는 system prompt 전체를 workflow에 넘기지 않고, `<env>`, `<available_skills>`, `Instructions from:` 블록만 추출한다.
- `build`, `plan`, `general`은 자체 prompt가 없으면 모델별 기본 system prompt를 공유할 수 있다.
- custom agent는 prompt와 tools를 자유롭게 바꿀 수 있으므로 built-in agent와 구분이 흐려진다.
- per-agent model override가 있더라도 model ID는 agent ID가 아니다.
- tools 목록은 permission 결과일 뿐이며 agent 이름의 정규 식별자가 아니다.

따라서 workflow가 agent별 분기나 로깅을 해야 한다면 명시 전달이 필요하다.

---

## 9. 권장 설계

가장 작은 변경은 opencode 쪽에서 provider request header에 agent 이름을 추가하고, adapter가 그 header를 읽어 workflow input으로 넘기는 방식이다.

권장 header:

```text
x-opencode-agent: build
```

추가로 agent mode까지 필요하면 다음 header를 별도로 둔다.

```text
x-opencode-agent-mode: primary
```

adapter 입력:

```ts
export interface RequestMeta {
  adapterRequestId: string;
  sessionId: string;
  userId: string;
  opencodeAgent?: string;
}

export interface WorkflowInputs {
  request_id: string;
  model: string;
  opencode_agent?: string;
  opencode_env: string;
  opencode_available_skills: string;
  opencode_instructions?: string;
  messages: string;
  messages_2: string;
  tools?: string;
  tool_choice?: string;
}
```

adapter handler에서는 `request.headers["x-opencode-agent"]`를 문자열로 정규화해 `requestMeta.opencodeAgent`에 넣고, `buildWorkflowInputs()`에서 값이 있을 때만 `inputs.opencode_agent`를 설정한다.

real AIU Workflow에서 이 값을 쓰려면 Start node에도 optional `opencode_agent` 변수를 추가해야 한다. workflow가 이 값을 쓰지 않는다면 adapter 내부 로그와 dump에만 보존해도 된다.

---

## 10. 구현 선택지

### 선택지 A: opencode plugin으로 header 추가

opencode의 `chat.headers` hook은 이미 `{ agent: input.agent.name }`을 받으므로, adapter 연동용 plugin에서 header를 추가할 수 있다. opencode 본체를 수정하지 않아도 되고, adapter 전용 운영 구성이 가능하다는 장점이 있다.

개념적 형태:

```ts
export default async function () {
  return {
    "chat.headers": async (input, output) => {
      output.headers["x-opencode-agent"] = input.agent
    },
  }
}
```

정확한 plugin export 형태는 현재 사용하는 opencode plugin 규약에 맞춰 확인해야 한다.

### 선택지 B: opencode core에서 기본 header로 추가

`packages/opencode/src/session/llm/request.ts`의 일반 provider header에 다음 값을 넣는 방식이다.

```ts
"x-opencode-agent": input.agent.name,
```

이 방식은 모든 provider에 agent header가 나가므로 adapter 외 provider에도 노출된다. core 변경으로 가져갈 때는 header 이름과 노출 범위를 명확히 합의해야 한다.

### 선택지 C: model ID로 우회

agent마다 다른 model ID를 설정하고 adapter가 model ID를 agent처럼 해석하는 방식이다. 구현은 쉽지만 설정 의존성이 커서 권장하지 않는다. agent가 model을 공유하거나 사용자가 model을 바꾸면 즉시 깨진다.

---

## 11. 최종 판단

현재 상태의 답은 "아니오"다. `aiu-opencode-adapter`는 opencode provider 요청에서 현재 agent 정보를 명시적으로 알 수 없다.

하지만 opencode는 provider request를 만들 때 `input.agent.name`을 이미 알고 있고 `chat.headers` hook에도 넘긴다. 따라서 `x-opencode-agent` header를 추가하고 adapter의 `RequestMeta`와 `WorkflowInputs`에 `opencode_agent`를 optional로 연결하면 안정적으로 해결할 수 있다.

agent별 workflow 분기, agent별 로그, subagent 품질 분석이 목적이라면 추론 대신 이 명시 전달 경로를 쓰는 것이 맞다.
