# OpenCode 명령어(CLI & Slash Commands) 개발자 가이드

이 가이드는 OpenCode가 제공하는 **명령행 명령어(CLI Subcommands)**, 대화창에서 사용하는 **슬래시 명령어(Slash Commands)**, 그리고 사용자 정의 명령어의 사용 방법과 별칭(Aliases) 체계를 상세히 정리한 개발자 참조 가이드입니다.

---

## 1. CLI 명령어 및 옵션 별칭 가이드

OpenCode CLI는 `yargs` 라이브러리를 기반으로 유연한 서브커맨드와 옵션 별칭(Alias)을 지원합니다. 터미널에서 `opencode <command>` 또는 단축형으로 명령어를 실행할 수 있습니다.

### 1) 핵심 소스 코드 레퍼런스
* **CLI 명령어 통합 디렉토리**: [packages/opencode/src/cli/cmd](file:///D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd)
  * 각 서브커맨드(`run`, `providers`, `mcp`, `plugin`, `agent` 등)의 정의 파일이 위치해 있습니다.
* **CLI 엔트리포인트**: [packages/opencode/src/cli/commands/commands.ts](file:///D:/AREA51/workspace/opencode/packages/cli/src/commands/commands.ts)
  * CLI 구동 시 전체 커맨드를 결합하여 빌드하는 핵심 엔트리 파일입니다.

---

### 2) CLI 명령어 및 별칭 요약표

| 주 명령어 (Command) | 명령어 별칭 (Aliases) | 서브커맨드 및 별칭 (Subcommands & Aliases) | 주요 옵션 별칭 (Option Aliases) | 설명 |
| :--- | :--- | :--- | :--- | :--- |
| **`run [message]`** | - | - | `-c` (`--continue`) <br>`-s <id>` (`--session`) <br>`-m <model>` (`--model`) <br>`-f <files>` (`--file`) <br>`-p <pw>` (`--password`) <br>`-u <user>` (`--username`) <br>`-i` (`--interactive`) | 지정한 질문이나 명령어를 실행하고 결과를 반환합니다. `-i` 옵션 적용 시 TUI 대화 모드로 진입합니다. |
| **`tui`** | - | - | `-c` (`--continue`) <br>`-s <id>` (`--session`) <br>`-m <model>` (`--model`) | TUI(터미널 사용자 인터페이스)를 즉시 실행합니다. |
| **`serve`** | - | - | - | 백그라운드 구동을 위한 headless OpenCode 서버를 시작합니다. |
| **`providers`** | **`auth`** | `list` (별칭: **`ls`**)<br>`login`<br>`logout` | `-p <provider>` (`--provider`) <br>`-m <model>` (`--model`) | AI 제공자(Anthropic, Gemini 등)의 크레덴셜 및 로그인 세션을 설정하고 검사합니다. (`opencode auth ls` 등 사용 가능) |
| **`mcp`** | - | `list` (별칭: **`ls`**)<br>`add`<br>`remove`<br>`auth list` (별칭: **`ls`**)<br>`auth login`<br>`auth logout` | - | MCP(Model Context Protocol) 서버 구성을 조작하고 인증을 관리합니다. (`opencode mcp ls`, `opencode mcp auth ls` 등 사용 가능) |
| **`plugin <module>`**| **`plug`** | - | `-g` (`--global`) <br>`-f` (`--force`) | 외부 npm 플러그인 모듈을 설치하고 자동 갱신 설정을 적용합니다. (`opencode plug <module> -g` 사용 가능) |
| **`agent`** | - | `create`<br>`list` | `-m <model>` (`--model`) <br>`--permissions` (별칭: **`--tools`**) | 커스텀 코딩 에이전트를 정의 및 빌드하거나 조회합니다. 에이전트 생성 시 `--tools` 옵션을 통해 허용할 권한 목록을 제어할 수 있습니다. |
| **`db`** | - | `$0 [query]` | `--format <json/tsv>` | 내장 SQLite 데이터베이스에 접근하여 원시 쿼리를 수행하거나 sqlite3 쉘을 실행합니다. |
| **`session`** | - | `list`<br>`delete <id>` | `-n` (`--max-count`) | 과거 대화 세션 기록을 탐색하거나 오래된 세션을 제거합니다. |
| **`stats`** | - | - | - | API 토큰 사용량 및 세션 런타임 지표 등 누적 통계를 출력합니다. |
| **`upgrade`** | - | - | `-m <model>` (`--model`) | OpenCode CLI 바이너리를 최신 버전으로 자동 업그레이드합니다. |
| **`uninstall`** | - | - | `-c` (`--config`) <br>`-d` (`--db`) <br>`-f` (`--force`) | 설정 파일, 세션 데이터베이스를 제거하고 CLI를 삭제합니다. |

---

## 2. 대화창 슬래시 명령어 (Slash Commands) 가이드

TUI 모드 또는 UI 웹 채팅 창의 프롬프트 입력 칸에서 `/` 접두사를 입력하면 등록된 슬래시 명령어를 빠르게 실행할 수 있습니다.

### 1) 빌트인 슬래시 명령어 (Built-in Commands)
이 명령어들은 시스템 레이아웃 및 세션 제어를 담당하며 내장 핸들러가 탑재되어 있습니다.

* **`/new`**: 새로운 대화 세션을 열고 화면을 초기화합니다. (`session.new`와 매핑)
* **`/undo`**: 바로 직전에 수행된 사용자 질문과 에이전트 답변 쌍(1 Turn)을 지우고, 당시 작성했던 사용자 질문 템플릿을 입력창에 도로 복원합니다.
* **`/redo`**: 되돌렸던(Undo) 대화 이력을 다시 되돌려 원상복구합니다.
* **`/compact`**: 현재까지의 긴 세션 내용을 강제로 요약 압축하여 LLM 컨텍스트 공간을 최적화합니다.
* **`/fork`**: 현재 대화 세션의 히스토리를 그대로 복제한 뒤, 별도의 세션 식별자를 생성하여 이어서 대화를 진행합니다.
* **`/share`**: 세션 내용의 전체 복사본 웹 공유 링크를 생성하고 클립보드에 바인딩합니다.
* **`/unshare`**: 현재 활성화된 세션의 기존 공유를 폐쇄하여 외부 접근을 차단합니다.
* **`/open`**: 파일 검색창 팝업을 열어 특정 소스 파일을 브라우징합니다.
* **`/terminal`**: 하단에 위치한 통합 개발 터미널 뷰를 켜거나 끕니다.
* **`/model`**: 실행 중인 세션에 할당된 LLM 모델을 전환할 수 있는 모델 피커 팝업을 표시합니다.
* **`/mcp`**: MCP 서버 연결을 켜거나 끄는 연동 모달 창을 엽니다.
* **`/agent`**: 등록된 여러 커스텀/빌트인 코딩 에이전트 간의 역할을 순환 전환합니다.
* **`/workspace`**: 현재 프로젝트 경로에 귀속된 Git 작업 트리의 변경 사항 파일 목록 뷰 패널을 토글합니다.

---

## 3. 사용자 정의 슬래시 명령어 (Custom Slash Commands) 설정

개발자는 본인 프로젝트에 최적화된 프롬프트나 작업을 단축어로 실행하기 위해 커스텀 슬래시 명령어를 생성하여 바인딩할 수 있습니다.

### 1) `opencode.json` 내 선언적 추가
`opencode.json`의 `"commands"` 프로퍼티에 명령어 이름과 템플릿을 키-값 쌍 객체로 선언합니다.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "commands": {
    "deploy": {
      "template": "Please build and deploy this application right now.",
      "description": "Builds and deploys the project with high reasoning"
    },
    "refactor": {
      "template": "Please analyze this file and propose structural refactoring improvements following clean code practices.",
      "description": "Refactor active files",
      "agent": "expert-coder",
      "subtask": true
    }
  }
}
```

#### 설정 프로퍼티 속성
* **`template`** (필수): 슬래시 커맨드를 호출했을 때 입력창에 채워지거나 에이전트에게 전송될 원시 프롬프트 본문입니다.
* **`description`** (선택): 슬래시 팝업창에서 명령어와 함께 사용자에게 노출될 가이드라인 설명 문구입니다.
* **`agent`** (선택): 해당 명령어를 처리할 전담 에이전트(예: `expert-coder`, `plan` 등)를 강제 매핑합니다.
* **`model` / `variant`** (선택): 명령어를 수행할 때 모델 및 파라미터 설정을 재정의합니다.
* **`subtask`** (선택): `true`인 경우 독립된 하위 태스크 브랜치로 분리 구동합니다.

---

### 2) 독립 마크다운 파일 기반 추가 (Workspace Commands Directory)
설정 파일이 비대해지는 것을 막기 위해 별도의 마크다운 문서로 분리 관리할 수도 있습니다.
* **경로 규격**: 프로젝트의 `./.opencode/command/<name>.md` 또는 `./.opencode/commands/<name>.md`
* **마크다운 형식**:
  ```markdown
  ---
  description: 분석 및 유닛 테스트 작성 자동화 명령
  agent: tester-agent
  ---
  Please read this code and generate comprehensive unit tests covering border cases.
  ```
  * 앞부분의 YAML Frontmatter 속성들이 설정을 결정하며, 마크다운 본문 텍스트가 전체 **`template`**으로 로드되어 작동합니다.
  * 파일명이 그대로 `/명령어` 이름이 됩니다 (예: `test-gen.md` -> `/test-gen`).
  * 수정 후 적용하려면 OpenCode 클라이언트를 **재시작**해야 합니다.
