# opencode 1.17.7 -> 1.17.9 System Prompt 변경 분석

> 비교 기준: `opencode` 1.17.7 로컬 분석 기준점 `dev@213ff3f2d` -> 현재 체크아웃 `v1.17.9@5c23e8841`
> 작성 목적: 1.17.7 분석 리포트의 system prompt 및 agent별 prompt 교체 분석이 1.17.9에서도 유효한지 확인

---

## 0. 결론

`REPORT.md`에 등록된 1.17.7 분석 내용 중 **agent에 따른 system prompt 교체 구조는 1.17.9에서도 그대로 유효하다.**

- `agent.prompt ? agent prompt : model provider prompt` 선택 로직은 변경되지 않았다.
- `build`, `plan`, `general`은 모델별 base prompt를 사용한다.
- `explore`, `title`, `summary`, `compaction`은 전용 agent prompt로 system 첫 블록을 교체한다.
- 실제 agent prompt 텍스트 파일(`packages/opencode/src/agent/prompt/*.txt`)은 1.17.7 대비 변경되지 않았다.
- session base prompt 텍스트 파일(`packages/opencode/src/session/prompt/*.txt`)도 `max-steps.txt` 삭제 외에는 변경되지 않았다.

따라서 AIU Workflow adapter 관점에서, agent별 system prompt 식별 및 Workflow template drift 위험에 대한 1.17.7 분석은 1.17.9에도 대부분 그대로 적용된다.

---

## 1. System prompt 조립 로직 변경 여부

### 1.1 Legacy request 경로

1.17.9에서도 system prompt의 첫 블록 선택은 동일하다.

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

근거:

- `packages/opencode/src/session/llm/request.ts`

의미:

- agent가 자체 prompt를 가지면 모델별 base prompt를 대체한다.
- agent prompt가 없으면 `SystemPrompt.provider(model)`가 모델 ID에 따라 base prompt를 선택한다.
- `<env>`, `Instructions from: ...`, `<available_skills>`는 base 뒤에 append된다.

### 1.2 모델별 base prompt 선택

`SystemPrompt.provider(model)`의 모델 ID 매칭 구조도 변경되지 않았다.

주요 선택 규칙:

- `gpt-4`, `o1`, `o3` 포함 -> `beast.txt`
- `gpt` 포함 -> `codex.txt` 또는 `gpt.txt`
- `gemini-` 포함 -> `gemini.txt`
- `claude` 포함 -> `anthropic.txt`
- `trinity` 포함 -> `trinity.txt`
- `kimi` 포함 -> `kimi.txt`
- 그 외 -> `default.txt`

AIU Workflow adapter가 노출하는 모델 ID가 위 패턴에 걸리지 않으면 계속 `default.txt`가 사용된다.

---

## 2. Agent별 system prompt 교체 변경 여부

### 2.1 Legacy agent 정의

`packages/opencode/src/agent/agent.ts`의 내장 agent prompt 매핑은 1.17.7 대비 변경되지 않았다.

| Agent | 1.17.9 동작 | 1.17.7 대비 |
| --- | --- | --- |
| `build` | 모델별 provider prompt 사용 | 변경 없음 |
| `plan` | 모델별 provider prompt 사용 | 변경 없음 |
| `general` | 모델별 provider prompt 사용 | 변경 없음 |
| `explore` | `agent/prompt/explore.txt`로 system base 교체 | 변경 없음 |
| `title` | `agent/prompt/title.txt`로 system base 교체 | 변경 없음 |
| `summary` | `agent/prompt/summary.txt`로 system base 교체 | 변경 없음 |
| `compaction` | `agent/prompt/compaction.txt`로 system base 교체 | 변경 없음 |
| custom agent | config `agent.<name>.prompt` 지정 시 base 교체 | 변경 없음 |

### 2.2 V2 agent plugin 정의

`packages/core/src/plugin/agent.ts`의 V2 내장 agent system 정의도 1.17.7 대비 변경되지 않았다.

V2 runner에서는 system prompt가 다음 형태로 조립된다.

```ts
system: [agent.info?.system, system.baseline]
  .filter((part): part is string => part !== undefined && part.length > 0)
  .map(SystemPart.make)
```

의미:

- V2에서도 agent별 system이 있으면 system context baseline 앞에 온다.
- `explore`, `title`, `summary`, `compaction`의 V2 system 문자열은 유지된다.
- 이번 업데이트는 agent별 system prompt 자체 변경이 아니라 step-limit 처리 변경에 가깝다.

---

## 3. 실제 prompt txt 파일 변경사항

다음 범위를 비교했다.

- `packages/opencode/src/agent/prompt/*.txt`
- `packages/opencode/src/session/prompt/*.txt`
- `packages/opencode/src/agent/generate.txt`
- `packages/opencode/src/command/template/*.txt`
- `packages/core/src/plugin/command/*`

결과:

| 파일 범위 | 변경 |
| --- | --- |
| `packages/opencode/src/agent/prompt/compaction.txt` | 없음 |
| `packages/opencode/src/agent/prompt/explore.txt` | 없음 |
| `packages/opencode/src/agent/prompt/summary.txt` | 없음 |
| `packages/opencode/src/agent/prompt/title.txt` | 없음 |
| `packages/opencode/src/session/prompt/default.txt` 등 base prompt | 없음 |
| `packages/opencode/src/command/template/initialize.txt` | 없음 |
| `packages/opencode/src/command/template/review.txt` | 없음 |
| `packages/opencode/src/session/prompt/max-steps.txt` | 삭제됨 |

`max-steps.txt`는 내용 변경이 아니라 위치 변경이다.

기존:

```text
packages/opencode/src/session/prompt/max-steps.txt
```

현재:

```text
packages/core/src/session/runner/max-steps.ts
```

현재 파일은 `MAX_STEPS_PROMPT` 상수로 같은 prompt 내용을 제공한다.

---

## 4. 1.17.7 분석에서 수정해야 할 항목

### 4.1 `MAX_STEPS` prompt 경로

1.17.7 분석 문서가 `MAX_STEPS` prompt 근거 파일을 다음처럼 설명했다면:

```text
packages/opencode/src/session/prompt/max-steps.txt
```

1.17.9 기준으로는 다음으로 수정해야 한다.

```text
packages/core/src/session/runner/max-steps.ts
```

prompt 내용 자체는 실질적으로 동일하다.

### 4.2 후속 user message의 `<system-reminder>` wrapping 제거

1.17.7의 legacy prompt loop에는 `step > 1 && lastFinished` 조건에서 새 user text part를 다음 형태로 감싸는 로직이 있었다.

```text
<system-reminder>
The user sent the following message:
...
Please address this message and continue with your tasks.
</system-reminder>
```

1.17.9에서는 이 wrapping 로직이 `packages/opencode/src/session/prompt.ts`에서 제거되었다.

영향:

- system prompt base 교체 규칙에는 영향이 없다.
- 다만 adapter가 `messages` 전체를 Workflow로 전달하는 경우, 후속 user steering 메시지의 model-facing 형태가 1.17.7 분석과 달라질 수 있다.

### 4.3 V2 runner step-limit 처리 변경

V2 runner에서는 hard-coded `MAX_STEPS = 25` 제한이 제거되고, agent별 `steps` 설정이 있을 때만 마지막 step 처리가 적용된다.

1.17.9 현재 동작:

- `agent.info?.steps`가 있고 `step >= agent.info.steps`이면 마지막 step으로 판단한다.
- 마지막 step에서는 tools를 materialize하지 않는다.
- `MAX_STEPS_PROMPT`를 assistant message로 추가한다.
- `toolChoice`를 `"none"`으로 설정한다.

근거:

- `packages/core/src/session/runner/llm.ts`

adapter 관점:

- tool call이 필요한 provider turn에서 마지막 step이면 tools가 비어 있고 `toolChoice: "none"`일 수 있다.
- 이는 agent별 system prompt 교체와는 별개지만, tool call 유도 프롬프트를 Workflow에서 별도로 관리하는 경우 마지막 step 요청을 구분할 필요가 있다.

---

## 5. AIU Workflow adapter 관점 영향

### 5.1 agent별 system prompt drift

이번 1.17.9 업데이트에서는 `explore`, `title`, `summary`, `compaction`의 prompt 텍스트가 바뀌지 않았다.

따라서 Workflow 내부에 1.17.7 기준으로 복제해 둔 agent별 prompt template이 있다면, 이번 버전 업데이트로 인한 즉각적인 문구 drift는 없다.

하지만 구조적으로는 여전히 drift 위험이 남아 있다.

- opencode가 agent prompt txt를 나중에 바꾸면 Workflow template은 자동 반영되지 않는다.
- adapter가 opencode 원본 system prompt를 Workflow에 전달하는 hybrid 전략을 쓰면 이 유지보수 비용을 줄일 수 있다.

### 5.2 system prompt 식별 전략

1.17.9에서도 다음 식별 전략은 유효하다.

- system prompt에 `<env>`가 있으면 정식 session prompt loop 요청으로 볼 수 있다.
- system 첫 줄이 `You are a file search specialist...`이면 `explore` subagent 요청으로 볼 수 있다.
- `You are a title generator...`, `Summarize what was done...`, `You are an anchored context summarization assistant...`는 hidden auxiliary agent 요청 식별에 계속 사용할 수 있다.

단, prefix 문자열 직접 비교는 장기적으로 취약하다. 더 안정적인 방향은 opencode에서 agent 정보를 명시적으로 전달하거나, adapter가 원본 system prompt를 Workflow context로 전달하고 Workflow bridge prompt만 별도 관리하는 방식이다.

### 5.3 messages 형태 변화

후속 user message wrapping 제거는 Workflow 입력 `messages`의 형태에 영향을 줄 수 있다.

1.17.7 분석 중 `step > 1` user message가 `<system-reminder>`로 감싸진다고 설명한 부분은 1.17.9 기준으로 더 이상 맞지 않는다.

---

## 6. 검증에 사용한 주요 diff

확인한 주요 변경 파일:

```text
M packages/opencode/package.json
M packages/opencode/src/session/prompt.ts
D packages/opencode/src/session/prompt/max-steps.txt
R packages/opencode/src/session/prompt/max-steps.txt -> packages/core/src/session/runner/max-steps.ts
M packages/core/src/session/runner/index.ts
M packages/core/src/session/runner/llm.ts
```

agent prompt 관련 파일에서는 의미 있는 변경이 없었다.

```text
packages/opencode/src/agent/prompt/compaction.txt
packages/opencode/src/agent/prompt/explore.txt
packages/opencode/src/agent/prompt/summary.txt
packages/opencode/src/agent/prompt/title.txt
```

session base prompt 관련 파일에서도 `max-steps.txt` 이동 외의 prompt 문구 변경은 없었다.

---

## 7. 결론

1.17.7 분석 리포트의 핵심 결론인 "agent별 system prompt 교체는 `agent.prompt` 또는 agent system 정의에 의해 발생한다"는 1.17.9에서도 유지된다.

수정이 필요한 부분은 다음으로 제한된다.

1. `max-steps.txt` 파일 경로가 core 상수로 이동했다.
2. legacy prompt loop의 후속 user message `<system-reminder>` wrapping 로직이 제거됐다.
3. V2 runner의 step limit이 hard-coded 25회가 아니라 agent별 `steps` 기반으로 바뀌었다.

AIU Workflow adapter의 system prompt 전달 전략을 검토할 때는, 이번 1.17.9 업데이트가 agent prompt template drift를 만들지는 않았지만, opencode 원본 system prompt를 활용하는 hybrid 전략의 필요성은 여전히 남아 있다고 보는 것이 타당하다.
