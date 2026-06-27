# opencode `task` 권한 deny 영향 분석

> 검토 대상: `D:\AREA51\workspace\opencode`
> 작성 일자: 2026-06-26
> 목적: `permission.task`를 `deny`로 설정했을 때 opencode의 일반 동작, subagent 실행, slash command 실행에 어떤 영향이 생기는지 정리한다.

---

## 1. 결론

`task` 권한을 `deny`로 설정해도 opencode 서버, 세션 루프, TUI, 일반 tool 실행 자체가 중단되지는 않는다.

영향 범위는 주로 **모델이 Task tool을 통해 subagent를 자동 호출하는 기능**이다. 즉 `general`, `explore` 같은 subagent로 작업을 위임하는 경로가 제한된다. 그 결과 복잡한 코드 탐색, 병렬 분석, 긴 출력 위임 처리의 효율이 떨어질 수 있지만, `read`, `grep`, `bash`, `edit`, `todowrite` 같은 다른 tool 권한에는 직접 영향이 없다.

다만 slash command 중 `subtask`로 실행되는 명령은 일반 Task tool 노출 여부와 다른 경로를 탄다. 기본 `/init`은 `subtask` 명령이 아니므로 `task: deny` 때문에 명령 자체가 막히지 않는다. 기본 `/review`는 `subtask: true`라서 서버가 내부적으로 child task를 만들지만, 이 경로는 `TaskTool.execute(..., bypassAgentCheck: true)`를 사용하므로 부모 agent의 `task` deny가 command 시작 자체를 막지는 않는다.

---

## 2. `task` 권한의 의미

`task`는 하위 에이전트를 실행하는 tool이다.

핵심 구현:

- `packages/opencode/src/tool/task.ts`
  - `const id = "task"`
  - 실행 전 `ctx.ask({ permission: "task", patterns: [params.subagent_type], ... })`로 대상 subagent 이름을 기준으로 권한을 확인한다.
  - deny가 아니면 `agent.get(params.subagent_type)`로 subagent를 찾고 child session을 생성 또는 재사용한다.
- `packages/opencode/src/tool/registry.ts`
  - Task tool 설명에 붙는 "Available agent types" 목록을 만들 때 `Permission.evaluate("task", item.name, agent.permission)`가 `deny`인 subagent를 제외한다.
- `packages/opencode/src/session/llm/request.ts`
  - LLM 요청 준비 시 `Permission.disabled(...)` 결과를 기준으로 disabled tool을 모델에게 노출하지 않는다.

따라서 `permission.task`는 두 층에서 작동한다.

1. Task tool 전체 노출 여부
2. Task tool이 살아 있을 때 호출 가능한 subagent 목록 및 실행 가능 여부

---

## 3. 설정별 동작 차이

### 3.1 Task tool 전체 비활성화

```json
{
  "permission": {
    "task": "deny"
  }
}
```

`Permission.fromConfig`는 문자열 값을 `pattern: "*"` 규칙으로 변환한다.

```ts
{ permission: "task", pattern: "*", action: "deny" }
```

`Permission.disabled`는 마지막으로 매칭되는 permission rule이 `pattern === "*"`이고 `action === "deny"`이면 해당 tool을 disabled로 본다. 이 경우 `task` tool은 LLM 요청의 tool 목록에서 빠진다.

효과:

- 모델은 Task tool을 볼 수 없다.
- 모델은 subagent를 자동 실행할 수 없다.
- Task tool 호출 자체가 모델 경로에서 거의 발생하지 않는다.
- 이미 stale하게 들어온 tool call은 unknown/stale tool 오류로 처리될 수 있다.

### 3.2 특정 subagent만 deny

```json
{
  "permission": {
    "task": {
      "*": "allow",
      "general": "deny"
    }
  }
}
```

이 경우 Task tool 자체는 유지될 수 있다. 대신 `general`은 Task tool 설명의 available agent 목록에서 제거되고, 실행 시도 시에도 `task/general` 권한 평가가 deny가 된다.

효과:

- Task tool은 모델에게 노출된다.
- 허용된 subagent만 안내된다.
- deny된 subagent는 모델이 시도하지 않도록 설명에서 빠진다.
- 모델이 그래도 호출하면 권한 오류로 실패한다.

### 3.3 기본 deny 후 일부 allow

```json
{
  "permission": {
    "task": {
      "*": "deny",
      "explore": "allow"
    }
  }
}
```

규칙은 마지막 매칭이 이긴다. 위 설정에서는 `explore`만 allow되고 나머지는 deny다.

주의할 점은 tool 전체 disabled 판정도 `findLast` 기반이라는 점이다. 마지막으로 매칭되는 `task` rule이 `explore allow`이면 Task tool 자체는 disabled가 아니며, Task 설명에는 deny가 아닌 subagent만 표시된다.

반대로 다음처럼 `* deny`가 마지막이면 Task tool 전체가 disabled 된다.

```json
{
  "permission": {
    "task": {
      "explore": "allow",
      "*": "deny"
    }
  }
}
```

---

## 4. opencode 일반 동작 영향

### 영향 없음 또는 낮음

- 세션 생성, 메시지 저장, title/summary/compaction 같은 내부 관리 흐름 자체
- `read`, `grep`, `glob`, `list`, `bash`, `edit`, `write`, `apply_patch`, `webfetch`, `websearch`, `skill` 등 다른 tool 권한
- slash command dispatch 자체
- TUI permission UI 자체

### 영향 있음

- 모델이 `explore` subagent에 코드 탐색을 위임하는 흐름
- 모델이 `general` subagent에 병렬 작업을 위임하는 흐름
- background subagent 실행
- 긴 tool output이 truncate됐을 때 "Task tool로 explore agent에게 처리시키라"는 안내
- agent prompt에 들어 있는 "Task tool을 적극 활용하라"는 지시의 실효성

`packages/opencode/src/tool/truncate.ts`는 agent가 `task/*`를 쓸 수 있으면 truncate 안내에 Task 위임을 추천하고, deny이면 Grep/Read 직접 사용 안내로 바꾼다.

---

## 5. `/init`, `/review`, custom command 영향

### 5.1 `/init`

기본 `/init` 정의는 `packages/opencode/src/command/index.ts`에 있다.

```ts
commands[Default.INIT] = {
  name: Default.INIT,
  description: "guided AGENTS.md setup",
  source: "command",
  get template() {
    return PROMPT_INITIALIZE.replace("${path}", ctx.worktree)
  },
  hints: hints(PROMPT_INITIALIZE),
}
```

여기에는 `subtask: true`가 없다. 따라서 `/init`은 기본적으로 command template을 현재 또는 기본 primary agent의 user prompt로 넣는 흐름이다. `task: deny`가 `/init` 실행 자체를 막지는 않는다.

다만 `/init` 수행 중 모델이 저장소 조사를 위해 `explore` subagent를 Task tool로 호출하려는 경우에는 영향을 받는다. `task: deny`이면 모델이 subagent 위임을 못 하므로 직접 `read`, `grep`, `glob`, `bash` 등을 사용하게 된다. 이 경우 작업은 가능하지만 탐색 효율과 context 사용 면에서 불리할 수 있다.

### 5.2 `/review`

기본 `/review`는 `subtask: true`다.

```ts
commands[Default.REVIEW] = {
  name: Default.REVIEW,
  description: "review changes [commit|branch|pr], defaults to uncommitted",
  source: "command",
  get template() {
    return PROMPT_REVIEW.replace("${path}", ctx.worktree)
  },
  subtask: true,
  hints: hints(PROMPT_REVIEW),
}
```

command 실행부는 다음 조건이면 `subtask` part를 만든다.

```ts
const isSubtask = (agent.mode === "subagent" && cmd.subtask !== false) || cmd.subtask === true
```

이후 session loop는 `subtask` part를 발견하면 `handleSubtask`를 실행한다. 이 경로는 `TaskTool.execute`를 내부적으로 호출하지만 `extra: { bypassAgentCheck: true, promptOps }`를 넘긴다.

의미:

- `/review` command 시작 자체는 `task` 권한 deny에 의해 막히지 않는다.
- 서버가 명령 처리 과정에서 만든 subtask 실행은 부모 agent의 Task tool 권한 체크를 우회한다.
- 하지만 실행된 child agent 내부에서 다시 Task tool로 다른 subagent를 호출하는 것은 child agent 권한에 따라 제한된다.

### 5.3 custom command

사용자 정의 command는 `command.<name>.agent`와 `command.<name>.subtask` 설정에 따라 동작이 달라진다.

- `agent`가 primary agent이거나 `subtask: false`이면 일반 prompt 흐름이다.
- `agent`가 subagent이고 `subtask !== false`이거나 command에 `subtask: true`가 있으면 서버가 `subtask` part를 만든다.
- 이 내부 subtask 시작은 `/review`와 같은 우회 경로를 탄다.
- 하지만 subtask 안에서 추가 Task tool 호출을 하는 것은 별도 권한 영향을 받는다.

---

## 6. 권장 설정

### 완전히 subagent 자동 위임을 막고 싶을 때

```json
{
  "permission": {
    "task": "deny"
  }
}
```

적합한 경우:

- 모든 작업을 parent agent가 직접 처리하기 원함
- child session 증가를 원하지 않음
- 비용 또는 병렬 실행을 강하게 제한하고 싶음

주의:

- `/init` 같은 조사형 작업이 느려질 수 있다.
- `/review` 내부 reviewer는 시작되지만, reviewer가 Explore agent로 패턴 확인을 위임하는 흐름은 막힐 수 있다.

### 탐색용 subagent만 허용하고 싶을 때

```json
{
  "permission": {
    "task": {
      "*": "deny",
      "explore": "allow"
    }
  }
}
```

적합한 경우:

- 쓰기 가능한 `general` delegation은 막고 싶음
- read-only 탐색 최적화는 유지하고 싶음
- `/init`, `/review` 품질 저하를 줄이고 싶음

### 수동 승인 기반으로 운영하고 싶을 때

```json
{
  "permission": {
    "task": {
      "*": "ask",
      "general": "deny",
      "explore": "allow"
    }
  }
}
```

적합한 경우:

- 대부분의 subagent 호출은 승인받고 싶음
- `explore`만 자동 허용하고 싶음
- `general`처럼 권한이 넓은 subagent는 금지하고 싶음

---

## 7. 최종 판단

`task` deny는 opencode 동작 자체를 망가뜨리는 설정이 아니다. 정확히는 **모델 주도 subagent delegation을 제한하는 권한 정책**이다.

운영 관점에서는 다음처럼 판단하면 된다.

| 설정 | opencode 자체 동작 | slash command | 모델 subagent 위임 | 권장 상황 |
| --- | --- | --- | --- | --- |
| `task: "deny"` | 정상 | `/init` 정상, `/review` 시작 정상 | 불가 | child session/병렬 실행을 전부 막고 싶을 때 |
| `task: { "*": "deny", "explore": "allow" }` | 정상 | 정상 | `explore`만 가능 | 안전한 탐색 위임은 유지하고 싶을 때 |
| `task: { "*": "ask" }` | 정상 | 정상 | 승인 필요 | 호출별 통제를 원할 때 |
| 기본값 | 정상 | 정상 | 대체로 가능 | opencode 기본 효율을 유지할 때 |

실무적으로는 완전 차단보다 `explore`만 허용하는 방식이 균형이 좋다. `explore`는 기본적으로 read-only 탐색에 특화되어 있고, `/init`이나 `/review` 같은 command의 조사 품질과 context 효율을 유지하는 데 도움이 된다.
