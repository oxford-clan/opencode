# opencode System Prompt 교체(replacement) 케이스 분석 보고서

> 분석 대상: `D:\AREA51\workspace\opencode`
> 계기: `/init` 실행 시 메시지 이력에 `role=system, content="You are a file search specialist..."` 가 실제로 관측됨 → system prompt가 교체되는 경로 재분석
> 작성 일자: 2026-06-17
> 관련: `opencode-system-prompt-management-report.md`, `opencode-init-agents-md-prompt-report.md`

---

## 0. 핵심 정정 및 결론

이전 보고서에서 `/init`을 "system prompt 교체가 아니라 command template(user 메시지) 주입"이라고만 설명한 것은 **불완전한 분석이었다.**

관측된 `role=system, content="You are a file search specialist..."`는 **`explore.txt`의 첫 줄**이다 (`packages/opencode/src/agent/prompt/explore.txt:1`). 즉:

> **`/init`을 실행하면, build 에이전트가 `task` 도구로 `explore` subagent를 호출하고, 그 subagent의 child 세션에서는 base system prompt가 `explore.txt`로 교체된다.** 사용자가 본 system 메시지는 이 child 세션의 것이다.

정리하면 `/init`은 **두 가지 일을 동시에** 일으킨다:
1. (main 세션) `initialize.txt` 템플릿이 **user 메시지**로 주입됨 — system은 평소대로 `default.txt`(+env+AGENTS.md).
2. (child 세션) build 에이전트가 codebase 조사를 위해 `task(subagent_type="explore")`를 호출 → **system prompt가 `explore.txt`로 교체된 별도 세션**이 생성됨.

따라서 "system prompt가 교체되는 케이스"는 분명히 존재하며, 그 핵심 트리거는 **subagent 호출**이다.

---

## 1. system prompt 교체가 일어나는 단 하나의 코드 지점

모든 교체는 `packages/opencode/src/session/llm/request.ts:58-66`의 이 한 줄로 수렴한다:

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    //   └ 에이전트가 자체 prompt를 가지면 그것으로 "교체"
    //                              └ 없으면 모델 ID 기반 기본 프롬프트
    ...input.system,                       // env + instructions(AGENTS.md) + skills  (= 추가, 교체 아님)
    ...(input.user.system ? [input.user.system] : []),
  ].filter((x) => x).join("\n"),
]
```

**규칙:**
- `input.agent.prompt`가 있으면 → 그 프롬프트가 **첫 블록(=base)을 완전히 대체**한다. (관측된 "You are a file search specialist"가 system 맨 앞에 온 이유.)
- 없으면 → `SystemPrompt.provider(model)`이 모델 ID로 base를 고른다 (`default/anthropic/gemini/...`).
- `input.system`(env, AGENTS.md instructions, skills)은 base 뒤에 **덧붙는다(추가)** — 교체가 아니다.

즉 "교체"의 본질은 **어떤 agent로 그 턴이 실행되는가**에 달려 있다.

---

## 2. 교체를 일으키는 경로 2가지

### 경로 ① Subagent 호출 (`task` 도구) — `/init`이 사용하는 경로 ★

`packages/opencode/src/tool/task.ts`:
- `task` 도구는 `subagent_type`으로 지정된 agent를 찾고(`agent.get(params.subagent_type)`, task.ts:116),
- **새 child 세션을 생성**하며 `agent: next.name`을 지정하고(task.ts:142-158),
- 그 child 세션에 대해 `ops.prompt({ sessionID: nextSession.id, agent: next.name, ... })`를 실행한다(task.ts:186-198).

child 세션의 턴도 동일한 `prompt.ts` 루프 → `request.ts prepare()`를 타므로, **`next.agent.prompt`(예: explore.txt)가 base를 교체**한다.

→ build/plan 같은 primary 에이전트가 `task`로 subagent를 부를 때마다 system prompt 교체가 발생한다.

### 경로 ② 보조 생성 호출 (title / summary / compaction / generate)

이들은 사용자 턴이 아니라 opencode 내부가 자동으로 거는 **전용 목적 호출**이며, 각자 자기 프롬프트를 system으로 쓴다(=교체). 도구는 모두 deny.

---

## 3. `/init` 실행 타임라인 (관측 사실 재구성)

```
[사용자가 /init 입력]
  │
  ▼  main 세션 (agent = build, 모델 = AIU → default.txt)
  system: default.txt + <env> + Instructions from: AGENTS.md (+ skills)
  user  : initialize.txt 전문 (${path}/$ARGUMENTS 치환)   ← command template
  │
  ▼  build 에이전트가 initialize.txt 지시("조사하라")를 따라 task 도구 호출
  task(subagent_type = "explore", prompt = "...탐색 요청...")
  │
  ▼  child 세션 생성 (agent = explore)
  system: explore.txt("You are a file search specialist...") ← ★ base 교체
          + <env> + Instructions from: AGENTS.md
          (skills 블록은 explore 권한에서 skill deny라 생략)
  user  : task 호출 시 전달된 탐색 프롬프트
  │
  ▼  결과를 main 세션으로 회수 → build가 AGENTS.md 작성/갱신
```

→ 사용자가 메시지 이력에서 본 `role=system, "You are a file search specialist"`는 위 **child 세션**의 system 메시지다. (main 세션의 system은 여전히 `default.txt`로 시작한다.)

---

## 4. "system prompt 교체" 전체 카탈로그

`agent.prompt`/`item.system`이 설정되어 base를 대체하는 모든 에이전트:

| Agent | 교체 프롬프트 | 트리거 | 비고 |
| --- | --- | --- | --- |
| `explore` | `agent/prompt/explore.txt` | `task(subagent_type="explore")` | **/init·일반 조사에서 사용. 관측된 케이스.** skill/edit 등 deny, read/grep/glob/web만 allow |
| `title` | `agent/prompt/title.txt` | 새 대화 제목 자동 생성 | hidden, 전 도구 deny, temp 0.5 |
| `summary` | `agent/prompt/summary.txt` | 세션 요약 | hidden, 전 도구 deny |
| `compaction` | `agent/prompt/compaction.txt` | 컨텍스트 압축 | hidden, 전 도구 deny |
| `Agent.generate` | `agent/generate.txt` | 설명→새 agent 정의 생성 (`agent.ts:378` `system=[PROMPT_GENERATE]`) | env 없이 단독 system |
| 사용자 정의 agent | config `agent.<name>.prompt` 또는 `.md` | 해당 agent 선택/`task` 호출 | 사용자가 prompt 지정 시 |

**교체가 아닌(=base 유지) 에이전트** — `agent.prompt` 미설정:
- `build`(default), `plan`, `general` → `SystemPrompt.provider(model)` 사용.
  - 주의: `task(subagent_type="general")`로 불러도 general은 자체 prompt가 없어 **`default.txt`가 system**이 된다(교체 아님).

**base 자체의 선택(모델별)** — 교체와는 다른 축이지만 system 첫 블록이 달라지는 경우:
- `default / anthropic / gemini / gpt / codex / beast / kimi / trinity`(+`copilot-gpt-5`) — 모델 ID 매칭 (`system.ts:25-39`). AIU 모델 → `default.txt`.

---

## 5. 교체(replace) vs 추가(append) 명확화

system 메시지 안에서 다음은 **교체가 아니라 base 뒤에 덧붙는 것**이다 (혼동 방지):

| 요소 | 위치 | 성격 |
| --- | --- | --- |
| `<env>` 블록 | base 다음 | 항상 추가 (`sys.environment`) |
| `Instructions from: AGENTS.md/CLAUDE.md` | env 다음 | 추가 (`instruction.system`) |
| `<available_skills>` | 그 다음 | 추가 (권한 있을 때) |
| `STRUCTURED_OUTPUT_SYSTEM_PROMPT` | 끝 | json_schema일 때 push |
| plan/build-switch/max-steps reminder | user/assistant 메시지 | system 아님, 메시지로 주입 |

즉 한 턴의 system은 `[교체 가능한 base] + [고정적으로 추가되는 env/instructions/skills]` 구조다.

---

## 6. aiu-opencode-adapter 관점 시사점

1. **adapter는 (sub)session마다 system 첫 블록이 다른 요청을 받는다.**
   같은 `/init` 작업이라도 main 세션 요청(system=`default.txt`...)과 explore subagent 요청(system=`explore.txt`...)이 **별개의 `POST /v1/chat/completions`**로 들어온다. adapter는 이를 구분하지 않고 proxy하므로 동작에는 문제 없으나, 로그/디버깅 시 "왜 system 내용이 요청마다 다른가"를 이 구조로 설명할 수 있다.

2. **정적 prefix 제거 로직의 견고성 점검 필요.**
   adapter는 base 프롬프트를 잘라내기 위해 `<env>` 태그(및 그 앞 "You are powered by the model named...")를 기준으로 동적 부분을 추출한다. 그런데 교체 케이스의 base(explore.txt/title.txt 등)는 `default.txt`와 내용이 전혀 다르다. 다행히 추출 기준이 **고정 문자열/태그**이지 특정 base 내용에 의존하지 않으므로 동작은 유지되지만, 아래 2-1을 확인해야 한다.

3. **(2-1) env 태그가 없는 호출 존재.**
   `title`/`summary`/`compaction`/`generate` 같은 보조 호출은 환경 블록 없이 자기 프롬프트만 system으로 보낼 수 있다(어댑터 캡처에서 title 요청은 `<env>` 없이 "You are a title generator"만 관측됨). adapter는 이미 `env_tag_missing` fallback(ADP-023: `opencode_env: ""`로 계속 진행)을 갖추고 있어 안전하다. 반면 explore subagent는 정식 루프를 타므로 `<env>`가 포함된다.

4. **AGENTS.md 전달 갭은 subagent에도 동일.**
   explore subagent의 system에도 `Instructions from: AGENTS.md` 블록이 들어가지만, adapter의 `extractOpenCodeBlocks()`는 `<env>`/`<available_skills>`만 추출하므로 **subagent 경로에서도 AGENTS.md 지시문은 workflow로 전달되지 않는다** (이전 보고서 §6 권고 그대로 적용).

---

## 7. 근거 파일 색인

| 관심사 | 위치 |
| --- | --- |
| system base 교체 지점 (`agent.prompt ? ... : provider()`) | `packages/opencode/src/session/llm/request.ts:58-66` |
| 관측된 프롬프트 원본 ("You are a file search specialist") | `packages/opencode/src/agent/prompt/explore.txt:1` |
| `task` 도구의 child 세션 생성·agent 지정 | `packages/opencode/src/tool/task.ts:116-198` |
| 내장 에이전트별 system/prompt·권한 정의 (V2) | `packages/core/src/plugin/agent.ts:125-205` |
| 내장 에이전트 정의 (V1) | `packages/opencode/src/agent/agent.ts:138-262` |
| `Agent.generate` (generate.txt 단독 system) | `packages/opencode/src/agent/agent.ts:366-399` |
| env/instructions/skills 조립(추가분) | `packages/opencode/src/session/prompt.ts:1327-1335` |
| `/init` 커맨드 template 배선 | `packages/opencode/src/command/index.ts:78-86`, `packages/core/src/plugin/command.ts:17-21` |
| 모델별 base 선택 | `packages/opencode/src/session/system.ts:25-39` |
