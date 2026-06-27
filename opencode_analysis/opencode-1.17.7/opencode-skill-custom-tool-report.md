# opencode Skill 과 Custom Tool 비교 분석 보고서

## 1. 요약

opencode에서 **Skill**과 **Custom tool**은 모두 에이전트 동작을 확장하지만, 목적과 실행 경계가 다르다.

- **Skill**은 `SKILL.md`로 작성하는 재사용 가능한 지침, 워크플로, 참고 자료 묶음이다. 모델이 필요하다고 판단할 때 내장 `skill` tool을 호출해 내용을 대화 컨텍스트에 주입한다.
- **Custom tool**은 모델이 직접 호출할 수 있는 실행 함수다. TypeScript/JavaScript로 tool 정의를 만들고, 필요하면 내부에서 임의 언어의 스크립트나 외부 API를 실행한다.

결론적으로, **"어떻게 판단하고 작업할지"를 알려주려면 Skill**, **"실제로 무엇을 실행하거나 계산하거나 외부 시스템을 조작할지"를 추가하려면 Custom tool**이 적합하다.

## 2. 핵심 차이

| 구분 | Skill | Custom tool |
| --- | --- | --- |
| 본질 | 지침과 워크플로 문서 | 모델이 호출하는 실행 함수 |
| 작성 형식 | Markdown `SKILL.md` + YAML frontmatter | `.ts` 또는 `.js` tool 정의 |
| 실행 여부 | 자체 실행 없음. 내용을 읽어 모델 컨텍스트에 주입 | `execute()` 함수가 실제 코드 실행 |
| 모델 노출 방식 | 처음에는 이름과 설명만 노출, 필요 시 `skill` tool로 본문 로드 | 매 턴 tool 정의와 JSON schema가 모델의 호출 가능 도구로 노출 |
| 입력 스키마 | 없음. `name`으로 특정 skill을 로드 | Zod 또는 JSON schema 기반 인자 정의 |
| 출력 | Markdown 지침, base directory, 샘플 파일 목록 | 문자열 또는 structured output, metadata, attachments |
| 권한 키 | `permission.skill` 아래에서 skill 이름으로 제어 | tool ID 자체가 permission key |
| 안전 경계 | 실행하지 않으므로 상대적으로 안전. 단, 후속 tool 사용을 유도할 수 있음 | 임의 코드 실행 가능. 권한, 입력 검증, 에러 처리가 중요 |
| 적합한 용도 | 코딩 규칙, 릴리즈 절차, 리뷰 기준, 도메인 지식, 반복 워크플로 | DB 질의, API 호출, 사내 시스템 연동, 계산, 자동화 실행 |

## 3. Skill 분석

### 3.1 개념

Skill은 opencode가 발견할 수 있는 "전문화된 작업 지침"이다. 모델은 시스템 컨텍스트나 `skill` tool 설명에서 사용 가능한 skill의 `name`과 `description`을 보고, 현재 작업과 맞으면 `skill({ name })`을 호출한다. 호출 결과는 다음 형태의 모델용 텍스트로 들어간다.

- `<skill_content name="...">`
- skill 본문
- skill base directory
- 상대 경로 해석 안내
- skill 디렉터리 안의 샘플 파일 목록

즉 Skill은 tool처럼 직접 실행되는 코드가 아니라, 모델의 판단과 작업 방식을 바꾸는 **지연 로드 문서형 컨텍스트**다.

### 3.2 파일 배치

기본 경로:

- 프로젝트: `.opencode/skills/<name>/SKILL.md`
- 프로젝트 호환 경로: `.opencode/skill/<name>/SKILL.md`
- 글로벌: `~/.config/opencode/skills/<name>/SKILL.md`
- 글로벌 호환 경로: `~/.config/opencode/skill/<name>/SKILL.md`
- Claude 호환: `.claude/skills/<name>/SKILL.md`, `~/.claude/skills/<name>/SKILL.md`
- Agents 호환: `.agents/skills/<name>/SKILL.md`, `~/.agents/skills/<name>/SKILL.md`

추가 경로와 원격 skill은 `opencode.json`에서 지정할 수 있다.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": [".opencode/skills", "../shared-skills"],
    "urls": ["https://example.com/.well-known/skills/"]
  }
}
```

원격 URL은 `index.json`을 제공하고, 각 skill 항목에 `SKILL.md` 및 참고 파일 목록을 포함하는 방식으로 캐시된다.

### 3.3 작성 예시

```markdown
---
name: release-notes
description: Use when preparing release notes or changelog entries from recent commits and PRs.
---

# Release Notes

When preparing release notes:

1. Inspect merged PRs and commits.
2. Group changes into features, fixes, docs, and chores.
3. Call out breaking changes explicitly.
4. Produce a concise changelog draft and a suggested version bump.
```

운영 관점에서는 `description`을 반드시 작성해야 한다. 현재 구현상 description 없는 skill도 내부 목록에는 들어갈 수 있지만, 모델에게 자동으로 노출되는 available skill 목록에서는 description 없는 항목이 빠진다. 따라서 description은 사실상 필수다.

### 3.4 권한과 노출 제어

Skill 로드는 `permission.skill`로 제어한다. 패턴은 skill 이름에 매칭된다.

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "ask",
      "secret-*": "deny"
    }
  }
}
```

특정 에이전트에서 skill tool 자체를 끌 수도 있다.

```json
{
  "agent": {
    "plan": {
      "tools": {
        "skill": false
      }
    }
  }
}
```

`deny`된 skill은 available skill 목록에서 숨겨지고 로드도 거부된다.

### 3.5 장점

- 구현 비용이 낮다. Markdown만 작성하면 된다.
- 코드 실행이 없어서 상대적으로 안전하다.
- 모델 컨텍스트를 효율적으로 쓴다. 전체 본문을 항상 넣지 않고, 이름과 설명만 먼저 노출한 뒤 필요할 때 로드한다.
- 워크플로, 정책, 리뷰 기준, 사내 규칙처럼 "판단 기준"을 재사용하기 좋다.
- skill 디렉터리의 참고 파일, 스크립트, 예시를 함께 묶을 수 있다.
- `.claude/skills`, `.agents/skills`와 호환되어 다른 에이전트 생태계의 지침을 일부 재사용할 수 있다.

### 3.6 단점

- 직접 실행 능력이 없다. API 호출, DB 질의, 파일 생성 같은 행동은 결국 다른 tool에 의존한다.
- 입력 스키마가 없으므로 구조화된 인자를 강제할 수 없다.
- 모델이 description을 보고 선택하므로 설명 품질이 낮으면 호출되지 않거나 잘못 호출될 수 있다.
- 내용이 너무 길면 로드 후 컨텍스트 비용이 커진다.
- "반드시 이 절차를 실행" 같은 강제성은 낮다. Skill은 지침이고, 실행 정책은 권한과 tool 쪽에서 별도로 관리해야 한다.

## 4. Custom Tool 분석

### 4.1 개념

Custom tool은 모델이 대화 중 직접 호출할 수 있는 함수다. opencode의 내장 `read`, `bash`, `edit` 같은 도구와 같은 ToolRegistry에 등록되고, 모델에는 tool description과 입력 JSON schema가 제공된다.

Tool 정의는 TypeScript 또는 JavaScript로 작성한다. 단, `execute()` 내부에서는 Python, shell, Go binary, HTTP API 등 어떤 구현도 호출할 수 있다.

### 4.2 파일 배치

도구 전용 경로:

- 프로젝트: `.opencode/tools/<name>.ts`
- 프로젝트 호환 경로: `.opencode/tool/<name>.ts`
- 글로벌: `~/.config/opencode/tools/<name>.ts`
- 글로벌 호환 경로: `~/.config/opencode/tool/<name>.ts`

플러그인에서도 tool을 등록할 수 있다.

- 프로젝트: `.opencode/plugins/*.ts`
- 글로벌: `~/.config/opencode/plugins/*.ts`
- `opencode.json`의 `plugin` 배열에 npm 패키지 또는 로컬 plugin 경로 지정

### 4.3 작성 예시

```ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query the project database with a read-only SQL statement.",
  args: {
    query: tool.schema.string().describe("Read-only SQL query to execute"),
  },
  async execute(args, context) {
    return `Would execute query in ${context.worktree}: ${args.query}`
  },
})
```

파일명이 tool 이름이 된다. 위 예시가 `.opencode/tools/database.ts`라면 tool ID는 `database`다.

한 파일에서 여러 named export를 내보내면 `<filename>_<exportname>` 형식으로 등록된다.

```ts
import { tool } from "@opencode-ai/plugin"

export const add = tool({
  description: "Add two numbers.",
  args: {
    a: tool.schema.number(),
    b: tool.schema.number(),
  },
  async execute(args) {
    return String(args.a + args.b)
  },
})
```

`.opencode/tools/math.ts`의 `add` export는 `math_add` tool이 된다.

### 4.4 Plugin 기반 등록

복잡한 확장에서는 plugin이 더 적합하다. 하나의 plugin에서 이벤트 hook과 custom tool을 함께 제공할 수 있다.

```ts
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async () => {
  return {
    tool: {
      project_status: tool({
        description: "Return project status from an internal service.",
        args: {
          project: tool.schema.string(),
        },
        async execute(args, context) {
          return `Status for ${args.project} in ${context.directory}`
        },
      }),
    },
  }
}
```

### 4.5 권한과 충돌

Custom tool은 tool ID를 기준으로 권한을 설정한다.

```json
{
  "permission": {
    "database": "ask",
    "project_status": "allow",
    "dangerous_deploy": "deny"
  }
}
```

구현상 custom tool은 내장 tool과 같은 이름을 사용할 수 있고, 같은 ID가 있으면 나중에 ToolRegistry 결과를 객체에 할당하는 과정에서 custom tool이 내장 tool보다 우선한다. 예를 들어 `.opencode/tools/bash.ts`를 만들면 내장 `bash`를 대체할 수 있다. 의도하지 않은 대체를 피하려면 고유한 이름을 쓰는 것이 안전하다.

### 4.6 장점

- 실제 코드를 실행할 수 있다.
- Zod 기반 인자 스키마로 모델 입력을 검증할 수 있다.
- 외부 API, DB, 사내 CLI, 빌드 시스템, 배포 시스템과 직접 연결할 수 있다.
- 결과를 문자열뿐 아니라 metadata, attachments와 함께 반환할 수 있다.
- plugin hook과 조합하면 tool 호출 전후 입력 수정, 로깅, 정책 적용이 가능하다.
- 내장 tool을 대체하거나 래핑할 수 있어 강한 커스터마이징이 가능하다.

### 4.7 단점

- 임의 코드 실행이 가능하므로 보안과 권한 설계가 중요하다.
- tool description과 schema가 부정확하면 모델이 잘못 호출하거나 인자를 잘못 만든다.
- 구현, 테스트, 배포, dependency 관리가 필요하다.
- startup 시 config directory의 dependency 설치와 동적 import에 영향을 받는다.
- 실패 처리를 잘못하면 모델 루프가 불안정해진다.
- 내장 tool과 이름이 충돌하면 의도치 않게 기본 동작을 바꿀 수 있다.

## 5. 적용 방법 비교

### 5.1 Skill 적용 절차

1. `.opencode/skills/<name>/SKILL.md` 생성
2. frontmatter에 `name`, `description` 작성
3. 본문에 작업 절차, 판단 기준, 예시, 금지 사항 작성
4. 필요하면 같은 디렉터리에 `references/`, `scripts/`, 예시 파일 배치
5. 민감한 skill은 `permission.skill`로 `ask` 또는 `deny` 설정
6. opencode 재시작

권장 작성 원칙:

- `description`은 "언제 사용할지"를 명확히 쓴다.
- Skill 본문은 실행 명령보다 판단 기준과 절차를 중심으로 쓴다.
- 상대 경로를 쓸 때는 skill base directory 기준임을 전제로 작성한다.
- 큰 참고 자료는 본문에 모두 붙이지 말고 별도 파일로 분리한다.

### 5.2 Custom tool 적용 절차

1. `.opencode/tools/<name>.ts` 생성
2. `@opencode-ai/plugin`의 `tool()` helper 사용
3. `description`과 `args` schema 작성
4. `execute(args, context)` 구현
5. 외부 dependency가 필요하면 `.opencode/package.json`에 추가
6. 권한을 `permission.<tool_id>`로 설정
7. opencode 재시작
8. 실제 세션에서 tool 호출 동작 검증

권장 구현 원칙:

- tool 이름은 내장 tool과 충돌하지 않게 짓는다.
- 인자는 좁고 명시적인 schema로 제한한다.
- 파일 경로는 `context.directory` 또는 `context.worktree` 기준으로 해석한다.
- 위험한 작업은 `context.ask()` 또는 opencode permission으로 승인 경계를 둔다.
- 반환 출력은 모델이 다음 행동을 결정할 수 있을 만큼만 간결하게 만든다.
- shell command를 실행한다면 escaping, timeout, read-only mode 같은 방어 장치를 둔다.

## 6. 선택 기준

| 상황 | 권장 |
| --- | --- |
| 팀 코딩 규칙, PR 리뷰 기준을 알려주고 싶다 | Skill |
| 릴리즈 절차, 장애 대응 절차, 운영 runbook을 재사용하고 싶다 | Skill |
| 특정 파일/프레임워크 작업 시 모델에게 사전 지식을 주고 싶다 | Skill |
| DB에서 값을 조회해야 한다 | Custom tool |
| Jira, GitHub, 사내 API를 호출해야 한다 | Custom tool 또는 MCP |
| 복잡한 계산이나 변환을 안정적으로 수행해야 한다 | Custom tool |
| 모델이 매번 같은 shell 절차를 안전하게 실행해야 한다 | Custom tool |
| 판단 기준도 필요하고 실행 자동화도 필요하다 | Skill + Custom tool 조합 |

## 7. 조합 패턴

Skill과 Custom tool은 경쟁 관계가 아니라 조합하면 효과적이다.

예시: 배포 자동화

- `deploy-runbook` Skill: 배포 전 체크리스트, 롤백 기준, 승인 정책, 실패 시 대응 절차를 설명
- `deploy_status` Custom tool: 현재 배포 상태 조회
- `deploy_start` Custom tool: 승인 후 배포 시작
- `deploy_rollback` Custom tool: 승인 후 롤백

이 구조에서는 Skill이 "언제 어떤 판단을 할지"를 담당하고, Custom tool이 "실제 시스템에 어떤 동작을 실행할지"를 담당한다.

## 8. 보안 관점

Skill은 직접 실행되지 않으므로 표면상 안전하지만, 모델에게 특정 tool 사용을 지시할 수 있다. 따라서 민감한 운영 절차나 내부 정보가 담긴 skill은 `permission.skill`로 제한하고, 실제 위험 행동은 별도 tool 권한에서 다시 막아야 한다.

Custom tool은 실행 코드이므로 보안 검토 대상이다.

- 입력 validation은 필수다.
- 외부 시스템 권한은 최소 권한으로 부여한다.
- destructive action은 기본 `ask` 또는 `deny`로 둔다.
- 로그에 secret을 남기지 않는다.
- 내장 `bash`, `edit`, `read`를 대체할 때는 의도와 영향 범위를 문서화한다.

## 9. 구현 근거 요약

확인한 주요 구현 위치:

- Skill discovery: `packages/opencode/src/skill/index.ts`
- 원격 skill download/cache: `packages/opencode/src/skill/discovery.ts`, `packages/core/src/skill/discovery.ts`
- 내장 skill tool: `packages/opencode/src/tool/skill.ts`, `packages/core/src/tool/skill.ts`
- Tool registry와 custom tool 로딩: `packages/opencode/src/tool/registry.ts`
- Plugin tool API: `packages/plugin/src/tool.ts`, `packages/plugin/src/index.ts`
- 문서: `packages/web/src/content/docs/skills.mdx`, `packages/web/src/content/docs/custom-tools.mdx`, `packages/web/src/content/docs/tools.mdx`, `packages/web/src/content/docs/plugins.mdx`

구현상 중요한 세부사항:

- `.opencode/skill`과 `.opencode/skills` 둘 다 스캔된다.
- `.opencode/tool`과 `.opencode/tools` 둘 다 스캔된다.
- Skill 본문은 처음부터 시스템 프롬프트에 모두 들어가지 않고, available skill 목록을 본 뒤 `skill` tool 호출로 로드된다.
- Skill tool은 skill 디렉터리의 파일 목록을 샘플링해 함께 보여준다.
- Custom tool은 `.opencode/tools/*.{js,ts}`에서 자동 로드되며, plugin의 `tool` hook에서도 등록된다.
- Custom tool의 Zod args는 JSON schema로 변환되어 모델에 전달된다.
- Custom tool 출력은 truncation 처리 후 모델에 반환된다.
- Tool 실행 전후에는 plugin hook `tool.execute.before`, `tool.execute.after`가 적용된다.

## 10. 권장안

opencode를 프로젝트에 적용할 때는 다음 기준을 권장한다.

1. 먼저 Skill로 팀 지식과 작업 절차를 문서화한다.
2. 반복 실행이 필요하거나 외부 시스템과 연결해야 하는 부분만 Custom tool로 만든다.
3. Skill에는 Custom tool의 사용 조건과 해석 방법을 적고, Custom tool에는 실행만 맡긴다.
4. 모든 Custom tool은 기본적으로 `ask`에서 시작하고, 충분히 검증된 read-only tool만 `allow`로 승격한다.
5. 내장 tool 이름을 대체하는 custom tool은 별도 문서와 테스트 없이는 피한다.
6. config, skill, plugin, tool 파일 변경 후에는 opencode를 재시작한다.

## 11. 최종 판단

Skill은 opencode 에이전트의 **지식과 절차를 확장하는 수단**이고, Custom tool은 에이전트의 **행동 능력을 확장하는 수단**이다.

좋은 운영 구조는 둘 중 하나만 고르는 것이 아니라, Skill로 "언제, 왜, 어떤 기준으로"를 정의하고 Custom tool로 "무엇을 안전하게 실행할지"를 구현하는 것이다. 이렇게 나누면 컨텍스트 비용, 보안 위험, 유지보수 부담을 각각 분리해 관리할 수 있다.
