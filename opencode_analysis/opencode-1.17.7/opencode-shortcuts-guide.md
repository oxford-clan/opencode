# OpenCode 단축키 및 커맨드 팔레트 개발자 가이드 (Shortcuts Guide)

이 문서는 OpenCode 사용자 및 개발자를 위해 단축키(Keybindings), 슬래시 커맨드(Slash Commands), 그리고 전역 커맨드 팔레트(Command Palette) 시스템의 전체 목록과 작동 메커니즘을 상세히 정리한 공식 가이드라인입니다.

---

## 1. 단축키 시스템 및 작동 메커니즘 개요

OpenCode의 단축키 엔진은 Solid.js 전역 컨텍스트를 활용해 실시간으로 관리됩니다.

### 1) 핵심 소스 코드 링크
* **단축키 컨텍스트 & 파서**: [packages/app/src/context/command.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/context/command.tsx)
  * `parseKeybind`, `matchKeybind`, `formatKeybind` 등의 유틸리티를 포함하며, 키 이벤트를 바인딩하고 가로챕니다.
* **대화 세션 내 명령어 등록**: [packages/app/src/pages/session/use-session-commands.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/pages/session/use-session-commands.tsx)
  * 세션 진행 중에 동작하는 대부분의 단축키와 슬래시 커맨드 핸들러가 포함되어 있습니다.
* **레이아웃 관련 명령어 등록**: [packages/app/src/pages/layout.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/pages/layout.tsx)
  * 사이드바, 워크스페이스, 테마, 프로젝트 관련 글로벌 레이아웃 레벨의 단축키를 등록합니다.
* **타이틀바 및 탭 관리 명령어 등록**: [packages/app/src/components/titlebar.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/components/titlebar.tsx)
  * 탭 전환, 탭 생성/닫기 및 히스토리 뒤로가기/앞으로가기 명령어가 들어있습니다.
* **프롬프트 입력 에디터 명령어 등록**: [packages/app/src/components/prompt-input.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/components/prompt-input.tsx)
  * 파일 첨부 단축키 및 에이전트 모드 전환(Normal, Shell) 관련 단축키가 선언되어 있습니다.

### 2) 입력 포커스 시 단축키 작동 예외 규칙 (Editable Target Safety)
사용자가 텍스트 입력창(`input`, `textarea`, `contenteditable`)에 무언가를 입력하고 있을 때 단축키가 무작위로 오작동하는 것을 방지하기 위해 다음과 같은 가드 로직이 적용되어 있습니다.
* 일반 문자/기호만 입력할 때는 단축키가 작동하지 않으며 본래의 키 동작이 정상 수행됩니다.
* 단, 수정키(`Ctrl`, `Cmd`, `Alt`)가 조합된 경우이거나, `Tab` 키인 경우, 전역 커맨드 팔레트 키인 경우, 혹은 예외로 허용된 아래의 **Editable Target Friendly 단축키**는 입력 포커싱 중에도 정상 작동합니다.
  ```ts
  // packages/app/src/context/command.tsx에 지정된 예외 목록
  const EDITABLE_KEYBIND_IDS = new Set([
    "terminal.toggle", // Ctrl+` (터미널 뷰 열기/닫기)
    "terminal.new",    // Ctrl+Alt+T (새 터미널 열기)
    "file.attach"      // Ctrl/Cmd+U (파일 첨부 다이얼로그)
  ])
  ```

### 3) OS 플랫폼별 매핑 규칙
* 설정 파일 및 코드 내에서 사용되는 **`mod`** 지시어는 운영체제에 따라 자동으로 변경됩니다.
  * **macOS / iOS**: `Command (⌘)` 키로 인식
  * **Windows / Linux**: `Control (Ctrl)` 키로 인식
* 예를 들어, `"mod+b"`는 Windows에서 `Ctrl+B`로, Mac에서는 `Cmd+B`로 렌더링되고 매칭됩니다.

---

## 2. 전역 단축키 & 명령어 종합 목록

아래 목록의 `mod` 표기는 OS에 맞춰 **`Ctrl`** 또는 **`Cmd`**로 대체하여 읽어주십시오.

### 1) 뷰 및 UI 패널 제어 (View & Panels)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 슬래시 커맨드 (Slash) | 설명 |
| :--- | :--- | :--- | :--- |
| **`sidebar.toggle`** | `mod+b` | - | 왼쪽 메인 사이드바를 열거나 닫습니다. |
| **`fileTree.toggle`** | `mod+\` | - | 사이드바 내 파일 탐색기 트리 뷰를 토글합니다. |
| **`terminal.toggle`** | `ctrl+`` (Backtick) | `/terminal` | 아래쪽 터미널 서랍(Drawer)을 열거나 닫습니다. |
| **`terminal.new`** | `ctrl+alt+t` | - | 새로운 쉘 터미널 세션을 즉시 생성합니다. |
| **`review.toggle`** | `mod+shift+r` | - | 코드 리뷰/피에르(Pierre) 리뷰 패널을 토글합니다. |
| **`common.goBack`** | `mod+[` | - | 내비게이션 히스토리 상 이전 화면으로 이동합니다. |
| **`common.goForward`**| `mod+]` | - | 내비게이션 히스토리 상 다음 화면으로 이동합니다. |
| **`input.focus`** | `ctrl+l` | - | 에디터 포커스가 어디에 있든 프롬프트 입력창으로 커서를 강제 이동시킵니다. |

### 2) 세션 및 대화 관리 (Session & Chat)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 슬래시 커맨드 (Slash) | 설명 |
| :--- | :--- | :--- | :--- |
| **`session.new`** | `mod+shift+s` | `/new` | 새로운 클린 대화 세션을 생성합니다. |
| **`session.undo`** | - | `/undo` | 마지막 한 턴(대화쌍)을 되돌리고, 직전 보냈던 프롬프트를 입력창에 다시 복원합니다. |
| **`session.redo`** | - | `/redo` | Revert(되돌리기)했던 마지막 대화를 다시 실행/복구합니다. |
| **`session.compact`** | - | `/compact` | 긴 대화 기록의 이전 컨텍스트를 서머리 압축(Compaction)하여 컨텍스트 윈도우 여유를 늘립니다. |
| **`session.fork`** | - | `/fork` | 현재 대화 상태를 그대로 복제하여 새 브랜치 세션을 생성합니다. |
| **`session.share`** | - | `/share` | 현재 대화 세션을 웹 공유 링크로 생성 및 클립보드 복사합니다. |
| **`session.unshare`** | - | `/unshare` | 기존 공유된 세션 링크를 비활성화(공유 중단)시킵니다. |
| **`session.archive`** | `mod+shift+backspace` | - | 현재 활성화된 대화 세션을 아카이브(보관함 이동) 처리합니다. |
| **`session.previous`**| `alt+arrowup` | - | 사이드바 목록 내 이전 세션으로 전환합니다. |
| **`session.next`** | `alt+arrowdown` | - | 사이드바 목록 내 다음 세션으로 전환합니다. |
| **`session.previous.unseen`**| `shift+alt+arrowup` | - | 읽지 않은 새로운 응답이 있는 이전 세션으로 즉시 이동합니다. |
| **`session.next.unseen`**| `shift+alt+arrowdown` | - | 읽지 않은 새로운 응답이 있는 다음 세션으로 즉시 이동합니다. |
| **`message.previous`**| `mod+alt+[` | - | 세션 본문 중 이전 사용자(User) 질문 메시지로 포커스를 이동합니다. |
| **`message.next`** | `mod+alt+]` | - | 세션 본문 중 다음 사용자(User) 질문 메시지로 포커스를 이동합니다. |

### 3) 파일 및 탭 편집 (Files & Tabs)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 슬래시 커맨드 (Slash) | 설명 |
| :--- | :--- | :--- | :--- |
| **`file.open`** | `mod+p` 또는 `mod+k` | `/open` | 프로젝트 내 파일을 빠르게 검색하여 열 수 있는 파일 팔레트 팝업을 엽니다. |
| **`file.attach`** | `mod+u` | - | 프롬프트 입력창에 파일 첨부(Context Attachment) 대화 상자를 엽니다. |
| **`tab.close`** | `mod+w` | - | 활성화된 현재 코드 편집 탭을 닫습니다. |
| **`tab.prev`** | `mod+option+arrowleft`| - | 이전 코드 탭으로 전환합니다. |
| **`tab.next`** | `mod+option+arrowright`| - | 다음 코드 탭으로 전환합니다. |
| **`tab.1` ~ `tab.9`** | `mod+1` ~ `mod+9` | - | 1번째부터 9번째 파일 편집 탭으로 빠르게 전환합니다. |
| **`context.addSelection`**| `mod+shift+l` | - | 탭 에디터 상에서 마우스 드래그로 선택된 라인 코드를 프롬프트 컨텍스트 첨부로 즉시 추가합니다. |
| **`workspace.new`** | `mod+shift+w` | - | 현재 프로젝트의 git worktree 등을 활용한 새 작업공간을 만듭니다. |
| **`workspace.toggle`** | - | `/workspace` | 사이드바의 워크스페이스 패널 영역 표시를 토글합니다. |

### 4) 프롬프트 입력 에디터 모드 (Input Modes)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 설명 |
| :--- | :--- | :--- |
| **`prompt.mode.shell`** | `mod+shift+x` | 터미널 명령어를 직접 작성하고 제출할 수 있는 쉘 실행(Shell) 모드로 전환합니다. |
| **`prompt.mode.normal`**| `mod+shift+e` | 일반 코딩 에이전트 자율 모드로 대화창 입력을 전환합니다. |

### 5) 프로젝트 이동 (Projects)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 설명 |
| :--- | :--- | :--- |
| **`project.open`** | `mod+o` | OpenCode 내 다른 등록된 프로젝트를 열기 위한 팝업을 노출합니다. |
| **`project.previous`**| `mod+alt+arrowup` | 이전 활성 프로젝트로 이동합니다. |
| **`project.next`** | `mod+alt+arrowdown` | 다음 활성 프로젝트로 이동합니다. |
| **`project.1` ~ `project.9`**| `mod+1` ~ `mod+9` | 지정 인덱스의 프로젝트를 활성화합니다. (구 버전 디자인 레이아웃에 한함) |

### 6) LLM 모델, 에이전트 및 MCP 제어 (Models & Agents)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 슬래시 커맨드 (Slash) | 설명 |
| :--- | :--- | :--- | :--- |
| **`model.choose`** | `mod+'` (Apostrophe) | `/model` | LLM 추론 모델 선택 모달 팝업을 실행합니다. |
| **`model.variant.cycle`**| `shift+mod+d` | - | 현재 모델의 사전 설정 옵션 파라미터 변형(Variant) 그룹을 순환 토글합니다. |
| **`agent.cycle`** | `mod+.` | `/agent` | 커스텀 에이전트 목록을 다음 에이전트로 한 단계 전환합니다. |
| **`agent.cycle.reverse`**| `shift+mod+.` | - | 커스텀 에이전트 목록을 역순으로 순환 전환합니다. |
| **`mcp.toggle`** | `mod+;` | `/mcp` | 연동되어 작동할 MCP(Model Context Protocol) 서버 구성 모달을 띄웁니다. |

### 7) 권한 통제 (Permissions)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 설명 |
| :--- | :--- | :--- |
| **`permissions.autoaccept`**| `mod+shift+a` | 현재 세션(또는 디렉토리)에 대해 도구 실행 권한 자동 수락(Auto Accept) 옵션을 On/Off 토글합니다. |

### 8) 전역 명령어 팔레트 (Command Palette)
| 기능명 (Command ID) | 기본 단축키 (Shortcut) | 설명 |
| :--- | :--- | :--- |
| **`command.palette`** | `mod+shift+p` | OpenCode가 지원하는 모든 등록된 단축키와 기능을 검색하여 즉시 실행할 수 있는 커맨드 센터 팔레트를 실행합니다. |

---

## 3. 사용자 지정 단축키 설정 방법 (Customization)

사용자는 디폴트로 할당된 단축키를 오버라이드하여 본인에게 맞는 커스텀 단축키를 바인딩할 수 있습니다.

### 1) `opencode.json`을 통한 단축키 커스텀 설정
단축키는 `opencode.json`의 `"keybinds"` 프로퍼티에 커맨드 ID별 키 조합 문자열을 선언하여 매핑합니다.
*(참고: `opencode.json` 수정 사항을 적용하려면 반드시 OpenCode 클라이언트를 **재시작**해야 합니다)*

```json
{
  "$schema": "https://opencode.ai/config.json",
  "keybinds": {
    "terminal.toggle": "ctrl+shift+t",
    "file.attach": "ctrl+u",
    "sidebar.toggle": "ctrl+e",
    "command.palette": "mod+shift+p"
  }
}
```

### 2) 키바인딩 표현 규칙
* 복수 조합 키는 플러스(`+`) 기호로 연결해 작성합니다 (예: `ctrl+shift+a`).
* 여러 개의 키 단축 단계를 체인 형태로 할당하고 싶을 때는 쉼표(`,`)를 활용하여 멀티 체인을 걸어줍니다 (예: `mod+k,mod+p` - VS Code 스타일 체인 바인딩).
* 특정 단축키 매핑을 아예 끄고 싶을 때는 `"none"` 문자열을 매핑합니다.
  ```json
  "keybinds": {
    "terminal.toggle": "none"
  }
  ```
