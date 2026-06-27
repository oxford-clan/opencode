# opencode `/init` AGENTS.md 생성 프롬프트 분석 보고서

> 분석 대상: `D:\AREA51\workspace\opencode`
> 출발점: 사용자 추정 — "`packages/core/src/plugin/agent.ts`가 `/init`로 최초 AGENTS.md를 만들 때 쓰는 프롬프트일 것"
> 작성 목적: 추정 검증 + `/init` 실제 동작 분석 + `aiu-opencode-adapter` 프로젝트에 적용할 점 검토
> 작성 일자: 2026-06-17

---

## 0. 결론 (TL;DR)

| 항목 | 결과 |
| --- | --- |
| 추정이 맞는가? | **아니오 (부분 오인).** `core/src/plugin/agent.ts`는 `/init` 프롬프트가 아니라 **내장 agent(build/plan/general/explore/compaction/title/summary)를 정의·등록하는 플러그인**이다. |
| `/init`의 실제 프롬프트는? | `packages/core/src/plugin/command/initialize.txt` (그리고 V1 사본 `packages/opencode/src/command/template/initialize.txt`). `command.ts`에서 `init` 커맨드의 `template`으로 연결되며 description은 `"guided AGENTS.md setup"`. |
| 왜 `/init`은 별도 프롬프트를 쓰나? | `/init`은 "코딩 작업"이 아니라 **"저장소를 조사해 다른 세션용 지침서(AGENTS.md)를 산출"하는 메타 작업**이라, 목적이 다른 전용 프롬프트가 필요하다. |
| 우리 프로젝트에 적용할 점은? | 직접 코드 이식은 없음. 다만 ① AGENTS.md 작성 철학 차용, ② `/init`을 실제로 돌려 어댑터 repo의 AGENTS.md 정비, ③ **이전 보고서와 연결되는 결정적 시사점**: `/init`이 잘 만든 AGENTS.md도 현재 adapter가 workflow로 전달하지 않는다 (§5). |

---

## 1. 추정 검증 — `agent.ts`의 정체

`packages/core/src/plugin/agent.ts`는 `PluginV2.define({ id: "agent" })` 형태의 **에이전트 정의 플러그인**이다. 부팅 시 `AgentV2.Service`에 내장 에이전트들을 등록/갱신한다 (`agent.ts:100-207`).

이 파일에 들어있는 문자열 상수들은 `/init`용이 아니라 **각 내장 에이전트의 system prompt**다:

| 상수 | 용도 | 적용 대상 |
| --- | --- | --- |
| `BUILD_SYSTEM` (agent.ts:12-13) | 기본 build agent의 fallback system (`item.system ??= BUILD_SYSTEM`) | `build` (default) |
| `PROMPT_EXPLORE` (15-31) | 파일 탐색 전문 서브에이전트 | `explore` |
| `PROMPT_COMPACTION` (33-41) | 대화 압축 요약 | `compaction` (hidden) |
| `PROMPT_TITLE` (43-86) | 대화 제목 생성 | `title` (hidden) |
| `PROMPT_SUMMARY` (88-98) | 세션 요약(PR 설명체) | `summary` (hidden) |

즉 `agent.ts`는 "어떤 에이전트가 있고, 각자 어떤 system prompt와 permission을 갖는지"를 정의하는 파일이다. AGENTS.md **파일 생성과는 무관**하다.

> **참고로 발견한 V1/V2 이중 구조**: 동일한 내장 에이전트 정의가 두 곳에 존재한다.
> - **V2(plugin 기반)**: `packages/core/src/plugin/agent.ts` — 프롬프트를 **코드 인라인 문자열**로 보유.
> - **V1**: `packages/opencode/src/agent/agent.ts` — 프롬프트를 `agent/prompt/*.txt` 파일에서 **import**.
>
> `PROMPT_TITLE` 등 내용은 양쪽이 사실상 동일하다. 어느 쪽이 런타임에 실제 사용되는지는 V2 세션 아키텍처 활성 여부에 따른다. (이전 보고서 `opencode-system-prompt-management-report.md`는 V1 경로 기준으로 작성했음 — 두 경로의 결론은 동일.)

---

## 2. `/init`의 실제 배선

`packages/core/src/plugin/command.ts`:

```ts
import PROMPT_INITIALIZE from "./command/initialize.txt"
...
editor.update("init", (command) => {
  command.template = PROMPT_INITIALIZE.replace("${path}", location.project.directory)
  command.description = "guided AGENTS.md setup"
})
```

- `/init` 슬래시 커맨드의 **template = `initialize.txt` 전문**이며, `${path}` 토큰만 현재 프로젝트 루트 경로로 치환된다.
- 같은 파일에서 `/review` 커맨드도 `review.txt`로 배선된다 (참고).
- V1 사본은 `packages/opencode/src/command/template/initialize.txt`에 있고, 내용은 거의 동일하나 **"repo-specific style or workflow conventions that differ from defaults"** 불릿 1개가 V1에만 추가되어 있다.

즉 `/init`은 별도 모델 호출이 아니라, **현재 에이전트가 위 지침서를 "사용자 프롬프트"처럼 받아 도구(read/glob/grep 등)로 저장소를 조사하고 AGENTS.md를 작성/갱신**하는 흐름이다.

---

## 3. `initialize.txt` 프롬프트 분석 (실제 `/init` 내용)

핵심 구조와 의도:

| 섹션 | 내용 | 설계 의도 |
| --- | --- | --- |
| 목표 | "Create or update `AGENTS.md`" — compact instruction file. 모든 줄이 *"Would an agent likely miss this without help?"* 를 통과해야 함 | **신호/잡음비 극대화**. AGENTS.md는 매 세션 system prompt에 통째로 주입되므로 짧고 고신호여야 한다 |
| `$ARGUMENTS` | 사용자 제공 focus/constraints 주입 | 사용자 의도 반영 |
| How to investigate | README/manifests/lockfiles → build·test·lint·typecheck·codegen config → CI/pre-commit → 기존 instruction 파일(AGENTS.md/CLAUDE.md/.cursorrules/copilot-instructions) → `opencode.json`. **"executable source of truth > prose"** | 추측·환각 방지, 검증 가능한 사실만 |
| What to extract | 정확한 dev 명령, 단일 테스트 실행법, **명령 순서**(lint→typecheck→test), monorepo 경계/엔트리포인트, toolchain quirks(codegen/migration/env 로딩), testing quirks(fixture/통합테스트 전제), 기존 제약 보존 | "여러 파일을 읽어야 알 수 있는 hard-earned context"만 |
| Questions | `question` 툴로 **한 배치만**, repo가 답 못하는 것(미문서화 관례, 브랜치/릴리스 규칙 등)만 | 자동화 친화, 불필요한 상호작용 최소화 |
| Writing rules | high-signal·repo-specific만 포함. **Exclude**: 일반 조언, 긴 튜토리얼, 자명한 언어 관례, 추측, `opencode.json` `instructions`로 분리하는 게 나은 내용. **"When in doubt, omit."** 짧은 섹션/불릿 | 문서 비대화 방지 |
| 갱신 규칙 | `${path}`에 AGENTS.md가 이미 있으면 **in-place 개선**(검증된 것 보존, 군더더기/낡은 주장 삭제, 현 코드와 reconcile) | 기존 지식 파괴 방지 |

---

## 4. 왜 `/init`은 일반 코딩 프롬프트가 아니라 이런 프롬프트로 "전환"하는가?

1. **작업 종류가 다르다.** 평소 system prompt(`session/prompt/default.txt` 등)는 "사용자의 코딩 작업을 수행"하는 에이전트 정체성이다. `/init`은 산출물이 **코드 변경이 아니라 메타-문서(AGENTS.md)**다. 목적이 다르므로 조사 절차·작성 규칙·금지 사항을 명시한 전용 작업 지시가 필요하다.
2. **AGENTS.md는 비용이 큰 영구 컨텍스트다.** 이전 보고서(`opencode-system-prompt-management-report.md` §4)에서 확인했듯, AGENTS.md 내용은 `instruction.system()`을 통해 **매 세션 system prompt에 `Instructions from:` 블록으로 주입**된다. 따라서 길고 잡다하면 토큰 낭비 + 주의 분산이 된다. `/init` 프롬프트가 "would an agent miss this?", "when in doubt omit"을 반복 강조하는 이유가 바로 이 주입 구조 때문이다.
3. **환각 억제.** "executable source > prose, 검증한 것만"은 LLM이 그럴듯한 가짜 명령/구조를 적는 것을 막는다.
4. **반복 실행 안전성.** in-place 개선 규칙은 `/init`을 두 번째 실행해도 기존의 잘 정리된 지침을 날리지 않게 한다.

---

## 5. `aiu-opencode-adapter` 프로젝트에 적용할 점

> 직접적인 코드 이식 대상은 없다 (`/init`은 opencode CLI 기능이지 어댑터가 구현할 것이 아니다). 다만 아래 세 가지가 실질적으로 유용하다.

### 5-1. (가장 중요) 이전 보고서와 이어지는 결정적 시사점
`/init`이 정성껏 만든 **AGENTS.md조차도, 현재 adapter는 workflow LLM으로 전달하지 않는다.**
- 근거 체인: `/init` → AGENTS.md 생성 → opencode가 `instruction.system()`으로 system prompt에 `Instructions from: ...AGENTS.md` **평문 블록**(태그 없음)으로 주입 → adapter의 `extractOpenCodeBlocks()`는 `<env>`/`<available_skills>` **태그만** 추출 → AGENTS.md 지시문은 누락.
- 즉 "AGENTS.md로 어댑터/workflow 동작을 제어"하려는 시도는 **현재 구조에서 무력화**된다.
- **권고**: `opencode-system-prompt-management-report.md` §6의 권고(=`Instructions from:` 블록도 추출해 `opencode_instructions` 파라미터로 전달, 또는 경계 문구 이후 전체 전송)를 채택해야 `/init` 산출물이 실제로 효력을 갖는다.

### 5-2. AGENTS.md 작성 철학 차용
어댑터 repo는 이미 루트/패키지별 `CLAUDE.md`·`AGENTS.md`와 `docs/` 체계를 갖췄다. `initialize.txt`의 원칙은 이들이 비대해지지 않게 하는 좋은 기준이다:
- "Would an agent likely miss this without help?" 필터로 줄 단위 검토.
- executable source(스크립트/config) 우선, 자명한 관례·일반 조언 배제, when-in-doubt-omit.
- 짧은 섹션/불릿, 기존 문서는 in-place 개선.

### 5-3. `/init`을 실제로 활용
어댑터 repo(`D:\AREA51\workspace\aiu-opencode-adapter`)에서 opencode CLI로 `/init`을 실행하면, 위 조사 절차에 따라 AGENTS.md 초안/개선안을 자동 생성할 수 있다. 단 §5-1 때문에, **그 결과가 workflow까지 도달하길 원한다면 adapter 전달 경로 보완이 선행되어야 한다.**

### 5-4. workflow LLM system prompt spec에도 동일 원칙 유효
어댑터의 `workflow-llm-system-prompt-spec.md`처럼 "고정 지침서"를 쓸 때도 initialize.txt의 "high-signal only / exclude generic / 명확한 출력 규칙" 접근은 그대로 참고가치가 있다.

---

## 6. 근거 파일 색인

| 관심사 | 위치 |
| --- | --- |
| 내장 agent 정의 플러그인(추정 대상, /init 아님) | `packages/core/src/plugin/agent.ts:100-207` |
| build agent fallback system 상수 | `packages/core/src/plugin/agent.ts:12-13` |
| explore/compaction/title/summary 프롬프트(인라인) | `packages/core/src/plugin/agent.ts:15-98` |
| **`/init` 실제 프롬프트(V2)** | `packages/core/src/plugin/command/initialize.txt` |
| `/init` 커맨드 배선 (template, `${path}` 치환, description) | `packages/core/src/plugin/command.ts:17-21` |
| `/init` 프롬프트 V1 사본 | `packages/opencode/src/command/template/initialize.txt` |
| AGENTS.md가 system prompt로 주입되는 경로(연계) | `packages/opencode/src/session/instruction.ts:60-169` (상세: `opencode-system-prompt-management-report.md`) |
| V1 내장 agent 정의(대조군) | `packages/opencode/src/agent/agent.ts` |
