# opencode System Prompt 관리 방식 분석 보고서

> 분석 대상: `D:\AREA51\workspace\opencode` (opencode 소스코드)
> 작성 목적: `aiu-opencode-adapter` 프로젝트가 opencode의 실제 system prompt 구성·전송 방식을 정확히 이해하기 위함
> 작성 일자: 2026-06-17

---

## 0. 요약 (TL;DR)

| 질문 | 답 |
| --- | --- |
| 사전 정의된 프롬프트가 소스에 존재하는가? | **존재한다.** `.txt` 파일로 빌드 시 번들에 포함된다 (`session/prompt/*.txt`, `agent/prompt/*.txt`). |
| 경우에 따라 다르게 설정되는가? | **그렇다.** ① 모델 ID별, ② agent별, ③ 요청 종류(title/summary/compaction)별, ④ 모드(plan/json_schema/max-steps)별로 달라진다. |
| AGENTS.md가 있으면 system prompt에 포함하는가? | **포함한다.** `instruction.system()`이 AGENTS.md / CLAUDE.md / 전역·설정 지정 파일을 읽어 system 메시지에 `Instructions from: <path>` 블록으로 삽입한다. 하위 디렉터리의 AGENTS.md는 파일을 read할 때 **tool 결과의 `<system-reminder>`로 동적 첨부**된다. |

**adapter 관점 핵심 시사점**: opencode가 보내는 system prompt는 `[모델별 베이스 프롬프트] + [<env> 블록] + [Instructions from: AGENTS.md/CLAUDE.md] + [<available_skills> 블록]` 구조다. 현재 adapter의 `extractOpenCodeBlocks()`는 `<env>`와 `<available_skills>` **태그만** 추출하므로, 그 사이에 있는 **프로젝트 AGENTS.md/CLAUDE.md 지시문은 workflow로 전달되지 않는다** (→ §6 참조).

---

## 1. System Prompt 조립 파이프라인

최종 조립은 `packages/opencode/src/session/prompt.ts`와 `session/llm/request.ts` 두 곳에서 일어난다.

### 1.1 조각 수집 — `prompt.ts:1327-1335`

```ts
const [skills, env, instructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),          // <available_skills> 블록
  sys.environment(model),     // <env> 블록 + available_references
  instruction.system().pipe(Effect.orDie),  // AGENTS.md / CLAUDE.md 등
  MessageV2.toModelMessagesEffect(msgs, model),
])
const system = [...env, ...instructions, ...(skills ? [skills] : [])]
if (format.type === "json_schema") system.push(STRUCTURED_OUTPUT_SYSTEM_PROMPT)
```

### 1.2 베이스 프롬프트 결합 — `session/llm/request.ts:58-66`

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,                                   // 위 1.1의 env+instructions+skills
    ...(input.user.system ? [input.user.system] : []),
  ].filter((x) => x).join("\n"),
]
```

즉 최종 system 텍스트의 논리적 순서는:

```
1. [agent.prompt] 또는 [모델별 베이스 프롬프트]   ← SystemPrompt.provider(model)
2. <env> ... </env>  (+ <available_references>)   ← sys.environment(model)
3. Instructions from: .../AGENTS.md\n<내용>        ← instruction.system()
   Instructions from: .../CLAUDE.md\n<내용>
   Instructions from: <config.instructions 항목>
4. Skills... <available_skills> ... </available_skills>  ← sys.skills(agent)
5. [user.system]  (있으면)
6. [STRUCTURED_OUTPUT_SYSTEM_PROMPT]  (json_schema 출력일 때)
```

이 전체가 `\n`으로 합쳐져 1개 문자열이 되고, 이후 ≤2개의 `role:"system"` 메시지로 prepend된다 (`request.ts:101-112`).

> **예외 경로** (`request.ts:99-112`):
> - OpenAI OAuth provider → system을 메시지가 아닌 `options.instructions`로 전달.
> - `isWorkflow === true` → system을 messages 앞에 붙이지 **않는다** (system은 별도 필드로만 보존).

---

## 2. Q1 — 사전 정의된 프롬프트가 소스에 존재하는가? → **존재함**

빌드 시 `.txt` 파일을 텍스트로 import해 바이너리에 포함한다 (`import PROMPT_X from "./prompt/x.txt"`).

### 2.1 모델별 베이스 프롬프트 — `packages/opencode/src/session/prompt/`

| 파일 | 첫 줄 특징 | 크기 |
| --- | --- | --- |
| `default.txt` | "You are opencode, an interactive CLI tool..." | ~8.5 KB / 95줄 |
| `anthropic.txt` | "You are OpenCode, the best coding agent on the planet." | ~8.2 KB / 105줄 |
| `gemini.txt` | "You are opencode, an interactive CLI agent specializing in..." | ~15.4 KB / 155줄 |
| `gpt.txt` | "You are OpenCode, You and the user share the same workspace..." | ~9.3 KB / 107줄 |
| `codex.txt` | "You are OpenCode, the best coding agent on the planet." | ~7.4 KB / 79줄 |
| `beast.txt` | "You are opencode, an agent - please keep going until..." | ~11 KB / 147줄 |
| `kimi.txt`, `trinity.txt`, `copilot-gpt-5.txt` | 모델 특화 변형 | — |

보조 프롬프트(같은 폴더): `plan.txt`, `plan-mode.txt`, `plan-reminder-anthropic.txt`, `build-switch.txt`, `max-steps.txt`.

### 2.2 내부 agent 프롬프트 — `packages/opencode/src/agent/prompt/`

| 파일 | 용도 |
| --- | --- |
| `title.txt` | 대화 제목 생성 ("You are a title generator. You output ONLY a thread title.") |
| `summary.txt` | 세션 요약 |
| `compaction.txt` | 컨텍스트 압축(compaction) |
| `explore.txt` | explore 서브에이전트 |

---

## 3. Q2 — 경우에 따라 다르게 설정되는가? → **그렇다 (4개 축)**

### 축 ① 모델 ID 별 — `session/system.ts:25-39`

```ts
export function provider(model: Provider.Model) {
  if (id.includes("gpt-4") || id.includes("o1") || id.includes("o3")) return [PROMPT_BEAST]
  if (id.includes("gpt")) return id.includes("codex") ? [PROMPT_CODEX] : [PROMPT_GPT]
  if (id.includes("gemini-")) return [PROMPT_GEMINI]
  if (id.includes("claude"))  return [PROMPT_ANTHROPIC]
  if (id.toLowerCase().includes("trinity")) return [PROMPT_TRINITY]
  if (id.toLowerCase().includes("kimi"))    return [PROMPT_KIMI]
  return [PROMPT_DEFAULT]
}
```

모델 ID 문자열 매칭으로 베이스 프롬프트가 선택된다. (매칭 안 되면 `default.txt`.)

> **adapter 시사점**: AIU workflow가 노출하는 모델 ID(`aiu/coding-agent` 등)는 위 어떤 패턴에도 안 걸리므로 opencode는 **`default.txt`**를 베이스 프롬프트로 사용한다. 즉 adapter가 받는 정적 prefix는 `default.txt` 내용이다.

### 축 ② Agent 별 — `agent/agent.ts` + `request.ts:60`

`input.agent.prompt`가 있으면 모델별 베이스 프롬프트를 **대체**한다.

| Agent | prompt 설정 | 베이스 프롬프트 |
| --- | --- | --- |
| `build` (기본), `plan`, `general` | 없음 | 모델별 `provider()` 사용 |
| `explore` | `PROMPT_EXPLORE` | 자체 프롬프트 |
| `title` (hidden) | `PROMPT_TITLE` | 자체 프롬프트 |
| `summary` (hidden) | `PROMPT_SUMMARY` | 자체 프롬프트 |
| `compaction` (hidden) | `PROMPT_COMPACTION` | 자체 프롬프트 |
| 사용자 정의 agent | config `agent.<name>.prompt` (`agent.ts:281`) | 지정 시 대체 |

### 축 ③ 요청 종류 별 (title / summary / compaction)

대화 제목 생성, 요약, 압축은 **별도의 작은 호출**이며 각각 전용 프롬프트 + `permission: { "*": "deny" }`(도구 비활성) + 낮은 temperature로 실행된다 (`agent.ts:232-262`). 이것이 캡처에서 관측된 "title generation 요청(=tools 없음)"의 정체다.

### 축 ④ 모드/포맷 별 추가 주입

- **plan 모드**: plan 관련 프롬프트/리마인더 주입.
- **json_schema 출력**: `STRUCTURED_OUTPUT_SYSTEM_PROMPT` 추가 (`prompt.ts:1335`).
- **마지막 step**: 어시스턴트 메시지로 `MAX_STEPS` 주입 (`prompt.ts:1343`).
- **후속 user 메시지**: step>1에서 `<system-reminder>`로 감싸 재주입 (`prompt.ts:1313-1320`).

### 축 ⑤ 항상 동적으로 붙는 `<env>` 블록 — `system.ts:61-72`

```
You are powered by the model named <api.id>. The exact model ID is <providerID>/<api.id>
Here is some useful information about the environment you are running in:
<env>
  Working directory: <cwd>
  Workspace root folder: <worktree>
  Is directory a git repo: yes|no
  Platform: <process.platform>
  Today's date: <date>
</env>
```

> **adapter 시사점**: adapter가 추출하는 `<env>` 태그 및 그 앞의 "You are powered by the model named..." 경계 문구가 바로 여기서 생성된다. (adapter의 ADP-016 구분자 `"You are powered by the model named"`의 출처.)

---

## 4. Q3 — AGENTS.md 처리 → **system prompt에 포함됨**

핵심 로직: `packages/opencode/src/session/instruction.ts`.

### 4.1 수집 대상과 우선순위 — `instruction.ts:60-153`

**전역 파일** (`globalFiles`, 먼저 존재하는 것 1개만 — `break`):
1. `{global.config}/AGENTS.md`
2. `~/.claude/CLAUDE.md`  *(단, `disableClaudeCodePrompt` 플래그가 없을 때만)*

**프로젝트 파일** (`instructionFiles`, cwd→worktree 상향 탐색, **첫 종류가 매칭되면 break**):
1. `AGENTS.md`
2. `CLAUDE.md`  *(`disableClaudeCodePrompt` 없을 때만)*
3. `CONTEXT.md`  *(deprecated)*

> 즉 `AGENTS.md`가 있으면 그것이 우선이고, 없을 때만 `CLAUDE.md`를 본다. 매칭된 종류에 대해서는 상위 디렉터리의 동일 파일들도 모두 추가한다("don't stack ... from every ancestor" 주석 — 종류 단위로만 break).

**설정 지정 파일** (`config.instructions`): 추가 glob/경로 + `http(s)://` URL(원격 fetch, 5초 타임아웃) — `instruction.ts:135-150, 158-167`.

`OPENCODE_DISABLE_PROJECT_CONFIG` 플래그가 켜지면 프로젝트 파일을 무시하고 `global.config`만 본다 (`instruction.ts:81-88, 123`).

### 4.2 system prompt에 들어가는 형태 — `instruction.ts:155-169`

```ts
return [
  ...paths.flatMap((item, i) => files[i] ? [`Instructions from: ${item}\n${files[i]}`] : []),
  ...urls.flatMap((item, i) => remote[i] ? [`Instructions from: ${item}\n${remote[i]}`] : []),
]
```

각 지시 파일은 `Instructions from: <절대경로>\n<파일 내용>` 형태의 **별도 문자열**이 되어, §1.1의 `instructions` 배열로 system에 합쳐진다. (태그로 감싸지 않는다.)

### 4.3 하위 디렉터리 AGENTS.md의 동적 첨부 — `instruction.ts:179-221` + `tool/read.ts:300,355-356`

opencode는 **파일을 read할 때** 그 파일이 위치한 디렉터리에서 루트까지 거슬러 올라가며 근처의 `AGENTS.md`/`CLAUDE.md`를 찾아(`resolve()`), 메시지당 1회씩 read 도구 결과 끝에 붙인다:

```ts
// tool/read.ts:355-356
if (loaded.length > 0) {
  output += `\n\n<system-reminder>\n${loaded.map(i => i.content).join("\n\n")}\n</system-reminder>`
}
```

즉 **상위(프로젝트 루트) AGENTS.md는 system prompt에**, **하위 디렉터리 AGENTS.md는 read 도구 결과(`role:"tool"` 메시지)의 `<system-reminder>`에** 들어간다. 이미 system에 포함된 파일이나 한 메시지에서 이미 첨부한 파일은 중복 제거된다(`resolve()`의 `sys.has/already.has/claims` 체크).

---

## 5. 최종 메시지 형태 (개념 예시)

`build` 에이전트 + 비-claude/gpt 모델(=`default.txt`) + 프로젝트 루트에 `AGENTS.md` 존재 시:

```
role: system
  You are opencode, an interactive CLI tool ...        ← default.txt 전문(정적)
  You are powered by the model named ...
  <env> Working directory: ... </env>                   ← 동적
  Instructions from: D:\proj\AGENTS.md
  <AGENTS.md 내용>                                       ← 프로젝트 지시문
  Skills provide specialized instructions ...
  <available_skills> ... </available_skills>            ← skill 권한 있을 때
role: user
  <사용자 메시지>
... (이후 turn에서 read 결과에 하위 AGENTS.md가 <system-reminder>로 추가될 수 있음)
```

---

## 6. adapter 프로젝트를 위한 시사점 / 권고

1. **정적 prefix의 정체 = `default.txt`**
   adapter가 콘텐츠 필터 회피를 위해 잘라내는 "정적 prefix"는 `session/prompt/default.txt`의 전문이다 (AIU 모델 ID가 어떤 패턴에도 안 걸려 `default.txt` 사용). opencode 버전이 올라가면 이 파일 내용이 바뀔 수 있으므로, 구분자/태그 기반 추출이 hash 고정보다 견고하다(현재 adapter가 이미 ADP-017 태그 방식으로 전환한 것은 올바른 선택).

2. **⚠️ 프로젝트 AGENTS.md/CLAUDE.md가 workflow로 전달되지 않는 갭**
   `<env>`/`<available_skills>`는 태그로 감싸여 있어 `extractOpenCodeBlocks()`가 추출하지만, **`Instructions from: ...AGENTS.md` 블록은 어떤 태그에도 들어있지 않아** adapter가 추출/전송하지 않는다. 사용자가 AGENTS.md로 동작을 지시해도 workflow LLM은 그 내용을 못 본다.
   - **권고 A**: adapter가 system prompt에서 `Instructions from:`로 시작하는 블록들도 추출해 별도 파라미터(예: `opencode_instructions`)로 workflow에 전달.
   - **권고 B(더 단순)**: 정적 베이스 프롬프트(경계 문구 이전)만 제거하고, 경계 문구부터 끝까지 전부를 workflow로 보내는 방식으로 변경하면 `<env>` + AGENTS.md + `<available_skills>`가 자연스럽게 함께 전달된다. (단, 이 방식은 과거 콘텐츠 필터 이슈 재현 여부를 검증해야 함.)

3. **하위 디렉터리 AGENTS.md는 이미 전달되고 있음**
   `resolve()`가 붙이는 하위 AGENTS.md는 `role:"tool"`(read 결과) 메시지의 `<system-reminder>`에 들어가고, adapter는 `messages` 전체를 직렬화해 전달하므로 **이 경로는 누락 없이 workflow에 도달**한다. 누락되는 것은 §6-2의 system-level 프로젝트 지시문뿐이다.

4. **title generation 분기는 모델 ID가 아니라 agent로 결정됨**
   title/summary/compaction은 tools 없는 별도 호출이며 전용 프롬프트를 쓴다. adapter는 이미 "tools 유무로 분기하지 않고 항상 proxy"하도록 ADP-018에서 정리했는데, 이는 opencode 구조와 일치한다(요청 종류 구분은 opencode 내부 책임).

5. **`isWorkflow` 경로 참고**
   `request.ts:102`의 `isWorkflow` 분기는 system을 메시지로 prepend하지 않는다. 이는 opencode 내부의 workflow provider 경로이며, AIU adapter(외부 OpenAI-compatible provider)와는 다른 경로다. adapter는 일반 provider 경로를 타므로 system이 정상적으로 `role:"system"` 메시지로 전달된다.

---

## 7. 근거 파일 색인

| 관심사 | 파일 / 위치 |
| --- | --- |
| 모델별 베이스 프롬프트 선택 | `packages/opencode/src/session/system.ts:25-39` |
| `<env>` / `<available_references>` 생성 | `packages/opencode/src/session/system.ts:55-92` |
| `<available_skills>` 생성 | `packages/opencode/src/session/system.ts:94-106` |
| 베이스+조각 결합, system→message 변환 | `packages/opencode/src/session/llm/request.ts:56-112` |
| 조각 수집·조립 호출부 | `packages/opencode/src/session/prompt.ts:1327-1347` |
| AGENTS.md/CLAUDE.md 수집·우선순위 | `packages/opencode/src/session/instruction.ts:60-169` |
| 하위 디렉터리 지시문 동적 해석 | `packages/opencode/src/session/instruction.ts:179-221` |
| read 결과에 지시문 `<system-reminder>` 첨부 | `packages/opencode/src/tool/read.ts:300, 355-356` |
| 내장/내부 agent 및 prompt 매핑 | `packages/opencode/src/agent/agent.ts:138-262, 281` |
| 사전 정의 프롬프트 텍스트 | `packages/opencode/src/session/prompt/*.txt`, `packages/opencode/src/agent/prompt/*.txt` |
