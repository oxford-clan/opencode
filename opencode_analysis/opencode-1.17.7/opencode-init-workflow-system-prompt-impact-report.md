# opencode `/init` 및 Agent별 System Prompt의 Workflow 전달 영향 검토

> 검토 대상: `D:\AREA51\workspace\opencode` 및 `D:\AREA51\workspace\aiu-opencode-adapter`
> 작성 일자: 2026-06-17
> 목적: `/init` 같은 명령과 subagent 실행 시 opencode system prompt가 달라지는 구조가 AIU Workflow LLM에 전달되는지, 누락 시 영향도와 방지 방안을 정리한다.

---

## 1. 결론

현재 adapter 구조에서는 opencode가 보내는 `role:"system"` 전체가 workflow로 전달되지 않는다. adapter는 system prompt에서 `<env>`와 `<available_skills>`만 추출해 `opencode_env`, `opencode_available_skills`로 보낸다.

따라서 `/init` 과정에서 main 세션에 들어가는 `initialize.txt`는 user 메시지로 전달되지만, `task(subagent_type="explore")` child 세션의 `explore.txt` base system prompt는 workflow에 전달되지 않는다. 또한 프로젝트 루트 `AGENTS.md`/`CLAUDE.md`는 system prompt의 `Instructions from:` 평문 블록으로 들어가므로 현재 adapter에서 누락된다.

가장 먼저 막아야 할 실질 리스크는 `AGENTS.md`/`CLAUDE.md` 누락이다. 이 문제는 `opencode_instructions` 파라미터를 추가하고 workflow LLM System Prompt 템플릿에 `{{opencode_instructions}}`를 삽입하면 해결된다.

---

## 2. opencode 쪽 동작

### 2.1 `/init` main 세션

`/init`의 실제 프롬프트는 `packages/core/src/plugin/command/initialize.txt` 및 V1 사본 `packages/opencode/src/command/template/initialize.txt`다.

이 프롬프트는 system prompt를 교체하지 않고, slash command template로 확장되어 main 세션의 user 메시지로 들어간다. adapter는 system이 아닌 메시지 이력을 `messages` JSON 문자열로 workflow에 보내므로 이 부분은 누락되지 않는다.

### 2.2 `/init` 중 explore subagent

`/init` 수행 중 build agent는 저장소 조사를 위해 `task` 도구로 `explore` subagent를 호출할 수 있다. 이 child 세션은 `packages/opencode/src/agent/prompt/explore.txt`를 base system prompt로 사용한다.

system 조립 지점은 `packages/opencode/src/session/llm/request.ts`의 다음 구조다.

```ts
...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
...input.system,
```

즉 `explore`, `title`, `summary`, `compaction`처럼 agent 자체 prompt가 있으면 모델별 base prompt가 해당 agent prompt로 교체된다.

### 2.3 AGENTS.md / CLAUDE.md

루트 지시문은 `packages/opencode/src/session/instruction.ts`에서 다음 형태로 system prompt에 추가된다.

```text
Instructions from: D:\path\AGENTS.md
<file content>
```

이 블록은 `<env>`나 `<available_skills>` 같은 태그로 감싸지지 않는다.

하위 디렉터리 지시문은 read 도구 결과 끝에 `<system-reminder>`로 붙으며, 이는 tool message content에 들어가므로 adapter의 `messages` 필드를 통해 workflow에 전달된다.

---

## 3. 현재 adapter 전달 범위

`aiu-opencode-adapter`의 `packages/adapter/src/workflow-client.ts` 기준 현재 추출 로직은 다음 범위만 다룬다.

```ts
const envMatch = fullSystemPrompt.match(/<env>([\s\S]*?)<\/env>/)
const skillsMatch = fullSystemPrompt.match(/<available_skills>([\s\S]*?)<\/available_skills>/)
```

현재 workflow 입력:

| 입력 | 전달 여부 | 비고 |
| --- | --- | --- |
| `/init` `initialize.txt` | 전달됨 | user 메시지로 `messages`에 포함 |
| `<env>` | 전달됨 | `opencode_env` |
| `<available_skills>` | 전달됨 | `opencode_available_skills` |
| 루트 `AGENTS.md` / `CLAUDE.md` | 누락 | `Instructions from:` 블록 미추출 |
| 하위 AGENTS.md system reminder | 전달됨 | tool message content로 `messages`에 포함 |
| `explore.txt` base prompt | 누락 | `<env>` 이전 base prompt라 미전송 |
| `title.txt` / `summary.txt` / `compaction.txt` | 누락 | 보조 agent system prompt가 미전송 |

---

## 4. 영향도

### 높음: 루트 AGENTS.md / CLAUDE.md 누락

프로젝트별 작업 규칙, 금지 사항, 응답 언어, 빌드/테스트 명령, 역할 경계가 workflow LLM에 보이지 않는다. 사용자가 AGENTS.md로 동작을 제어한다고 생각해도 실제 workflow LLM은 이를 따를 수 없다.

`aiu-opencode-adapter` 프로젝트 자체도 루트 `AGENTS.md`/`CLAUDE.md`에 handoff, role boundary, task ID 같은 작업 규칙을 두고 있으므로 실제 진행 품질에 직접 영향이 있다.

### 중간: `/init` explore subagent prompt 누락

`/init` 자체의 조사 지시는 user 메시지로 전달되므로 기능이 완전히 막히지는 않는다. 그러나 child 세션의 "file search specialist" 정체성, 편집 금지, 탐색 중심 행동 같은 base system 제약은 workflow LLM에 전달되지 않는다.

결과적으로 `/init`이 생성하는 AGENTS.md 품질이 opencode native provider 대비 낮아질 수 있다. 특히 조사 범위, 편집 회피, 결과 압축 방식에서 차이가 날 수 있다.

### 낮음~중간: title / summary / compaction 보조 호출

이 호출들은 UX 품질에 영향을 준다. 예를 들어 title generator system prompt가 누락되면 대화 제목이 길거나 일반 답변처럼 나올 수 있다. 다만 핵심 tool loop나 파일 변경 안정성보다는 영향이 낮다.

---

## 5. 누락 방지 방안

### 5.1 1단계: `opencode_instructions` 추가

우선 구현해야 할 방안이다.

adapter:
- system prompt에서 `Instructions from:` 블록을 추출한다.
- `WorkflowInputs`에 `opencode_instructions?: string`을 추가한다.
- 블록이 있을 때만 workflow inputs에 포함한다.

workflow-sandbox:
- `WorkflowInputs` 타입에 `opencode_instructions?: string`을 추가한다.
- LLM System Prompt template에서 `</env>` 이후, skills 이전에 주입한다.

real AIU Workflow:
- Start 노드에 optional `opencode_instructions` Paragraph 변수를 추가한다.
- LLM 노드 System Prompt에 `{{opencode_instructions}}`를 삽입한다.

이 변경은 `aiu-opencode-adapter` 문서에 이미 `ADP-053`, `SBX-030`, `WFL-004`로 등록되어 있다.

### 5.2 2단계: agent-specific prompt 보존 여부 결정

`explore.txt`, `title.txt`, `summary.txt`, `compaction.txt`까지 반영하려면 별도 설계가 필요하다.

가능한 방식:

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| base prompt 전문을 `opencode_agent_prompt`로 전달 | opencode 의미를 가장 많이 보존 | 과거 콘텐츠 필터 이슈 재발 가능, 토큰 증가 |
| agent 종류만 `opencode_agent_kind`로 전달 | 안전하고 짧음 | opencode prompt 세부 문구는 보존 안 됨 |
| allowlist 기반 짧은 요약을 `opencode_agent_context`로 전달 | 안전성과 품질의 균형 | adapter에 agent 판별 규칙 필요 |

권장안은 `opencode_agent_kind` 또는 짧은 `opencode_agent_context`다. 예를 들어 base prompt 첫 줄이나 known phrase로 `explore`, `title`, `summary`, `compaction`, `default`를 분류하고 workflow pre-system context에서 해당 목적을 반영한다.

---

## 6. Workflow Pre-system Context 변경 필요 여부

### `opencode_instructions`만 처리하는 경우

필요하다. 전체 prompt를 갈아엎을 필요는 없지만, workflow LLM System Prompt template에 placeholder 한 줄을 추가해야 한다.

```text
<env>
{{opencode_env}}
</env>
{{opencode_instructions}}
Skills provide specialized instructions and workflows for specific tasks.
```

이 위치는 opencode 원본 system prompt 순서(`env -> instructions -> skills`)와 일치한다.

### agent-specific prompt까지 처리하는 경우

추가 변경이 필요하다. 예를 들어 아래처럼 별도 placeholder를 둘 수 있다.

```text
{{opencode_agent_context}}
```

이 경우 workflow pre-system context에는 다음 우선순위도 명시해야 한다.

- `opencode_agent_context`는 해당 요청의 목적과 agent 역할을 설명한다.
- Tool call 출력 형식과 JSON 계약은 기존 Tool Use Protocol이 계속 우선한다.
- `opencode_instructions`는 프로젝트/사용자 지시문으로, 안전 정책과 출력 계약을 깨지 않는 범위에서 따라야 한다.

---

## 7. 권장 진행 순서

1. `ADP-053` 구현: `Instructions from:` 추출 및 `opencode_instructions` 전달.
2. `SBX-030` 구현: sandbox LLM template에 `{{opencode_instructions}}` 반영.
3. `WFL-004` 적용: 실제 AIU Workflow Start 노드 및 LLM 노드 수정.
4. E2E 검증: AGENTS.md에 명확한 지시를 넣고 workflow 응답 반영 여부 확인.
5. 후속 설계: `/init` explore 품질이나 title/summary 품질 문제가 남으면 `opencode_agent_kind` 또는 `opencode_agent_context`를 별도 task로 추가.

---

## 8. 최종 판단

현재 진행에 직접 영향을 주는 누락은 `AGENTS.md`/`CLAUDE.md` 전달 누락이다. 이는 사용자와 프로젝트가 기대하는 작업 규칙을 workflow LLM이 보지 못하게 하므로 우선순위가 높다.

`/init` 및 subagent별 base prompt 누락은 기능 차단보다는 품질 저하 리스크에 가깝다. 먼저 `opencode_instructions`를 닫고, 실제 `/init` E2E 결과에서 품질 문제가 확인되면 agent-specific context 전달을 후속으로 설계하는 것이 가장 안전하다.
