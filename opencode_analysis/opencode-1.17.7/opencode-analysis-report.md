# opencode 소스 분석 리포트

## 1. 분석 요약

opencode는 단순한 CLI가 아니라, **터미널 중심의 AI 코딩 에이전트 플랫폼**입니다. 소스 기준으로 보면 구조는 크게 4개로 나뉩니다.

- **CLI 진입/명령층**: [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1), [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L1)
- **TUI 인터랙션층**: [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L1), [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L1), [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L1)
- **Agent 런타임층**: [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L1), [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L1)
- **UI/렌더링 공용층**: [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1), [packages/ui/src/components/markdown.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L241), [packages/ui/src/components/session-diff.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/session-diff.ts#L1)

즉, opencode는 “터미널에서 한 번 프롬프트를 보내는 도구”가 아니라, **세션 지속성, 권한 승인, 파일 diff, 모델/provider 추상화, 플러그인 확장, 서버/attach 모드**까지 포함한 복합 제품입니다.

Claude Code, Codex CLI, Aider와 비교하면:

- Claude Code / Codex CLI처럼 **터미널에서 에이전트와 대화하는 UX**를 제공하지만
- Aider처럼 단순 git/diff 보조 도구에 머물지 않고
- **세션 저장, 재개, 포크, 공유, TUI 플러그인, 서버 attach, desktop/web/sdk**까지 갖춘 쪽이라
- 성격상 **“CLI 도구”라기보다 “agent platform”**에 가깝습니다.

## 2. 프로젝트 개요

### 목적
소스상 목적은 “AI-powered development tool”이며, 실제 구현은 **프로젝트 작업을 위한 대화형 코딩 에이전트**입니다. 루트 설명은 [package.json](D:/AREA51/workspace/opencode/package.json#L1), [README.md](D:/AREA51/workspace/opencode/README.md)에서 확인됩니다.

### 주요 사용 시나리오
- 새 세션을 열고 코드 수정 요청하기
- 파일 읽기/검색/패치/쓰기/쉘 실행을 에이전트에게 시키기
- 승인 대기 질문에 응답하기
- 긴 diff를 검토하고 되돌리기
- 이전 세션을 재개하거나 fork하기
- attach 모드로 실행 중인 서버에 붙기
- 플러그인/외부 provider를 통해 모델과 기능을 확장하기

### 제품 성격
- CLI 도구: `opencode run`, `opencode serve`, `opencode session` 같은 명령이 존재
- TUI 앱: 실제 상호작용은 SolidJS + OpenTUI 기반 화면이 중심
- Agent 런타임: 세션, 컨텍스트, tool call, approval, model turn이 내부 코어에서 돌음
- 복합 플랫폼: desktop/web/server/sdk까지 workspace에 포함

### 사용자 흐름
1. `opencode` 실행
2. config / plugin / provider / model 로딩
3. 홈 또는 세션 화면 진입
4. prompt 입력, 파일 첨부, slash command, autocomplete 사용
5. session create 또는 resume
6. agent turn 실행
7. tool 승인/실행
8. diff와 응답을 TUI에서 검토
9. 재개, 포크, 공유, 되돌리기 가능

## 3. Repository 구조

아래 표는 핵심 구조만 추렸습니다.

| 경로 | 역할 | 중요도 |
|---|---|---|
| [package.json](D:/AREA51/workspace/opencode/package.json#L1) | Bun workspace, Turbo scripts, 루트 빌드/개발 설정 | 상 |
| [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1) | CLI 진입점, yargs 명령 라우팅 | 상 |
| [packages/opencode/src/cli/bootstrap.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/bootstrap.ts#L1) | `InstanceRuntime` 로딩/정리 | 상 |
| [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L1) | `run` 명령의 핵심 실행 흐름 | 상 |
| [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L1) | TUI 루트, provider stack, render root | 상 |
| [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L1) | 프롬프트 입력/첨부/submit | 상 |
| [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L1) | slash/mention autocomplete | 중 |
| [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L1) | 세션 타임라인 화면 | 상 |
| [packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx#L1) | diff viewer 플러그인 | 중 |
| [packages/opencode/src/cli/cmd/tui/plugin/runtime.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/plugin/runtime.ts#L1) | TUI 플러그인 runtime/slots | 중 |
| [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L1) | location-scoped DI 그래프 | 상 |
| [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L1) | agent loop 오케스트레이션 | 상 |
| [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L1) | prompt admit/promote, steer/queue | 상 |
| [packages/core/src/session/context.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context.ts#L1) | 세션 히스토리 로드 | 상 |
| [packages/core/src/session/context-epoch.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context-epoch.ts#L1) | system context baseline/snapshot/revision | 상 |
| [packages/core/src/system-context.ts](D:/AREA51/workspace/opencode/packages/core/src/system-context.ts#L1) | 시스템 컨텍스트 추상화 | 상 |
| [packages/core/src/instruction-context.ts](D:/AREA51/workspace/opencode/packages/core/src/instruction-context.ts#L1) | `AGENTS.md` ambient instruction 로드 | 상 |
| [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L1) | global/project config 로더 | 상 |
| [packages/core/src/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/provider.ts#L1) | provider identity/config abstraction | 상 |
| [packages/core/src/model.ts](D:/AREA51/workspace/opencode/packages/core/src/model.ts#L1) | model metadata/capability/cost | 상 |
| [packages/core/src/catalog.ts](D:/AREA51/workspace/opencode/packages/core/src/catalog.ts#L1) | provider/model registry | 상 |
| [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L1) | tool registry/settlement | 상 |
| [packages/core/src/tool/builtins.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/builtins.ts#L1) | built-in tool 목록 | 상 |
| [packages/ui/src/components/*](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L241) | 공용 렌더러(마크다운/디프/메시지) | 중 |
| [packages/app](D:/AREA51/workspace/opencode/packages/app) | 웹 앱 | 중 |
| [packages/desktop](D:/AREA51/workspace/opencode/packages/desktop) | Electron 데스크톱 앱 | 중 |
| [packages/server](D:/AREA51/workspace/opencode/packages/server) | 서버/API | 중 |
| [packages/sdk/js](D:/AREA51/workspace/opencode/packages/sdk/js) | JS SDK | 중 |

패키지 매니저는 Bun이고, workspace는 `packages/*`, `packages/console/*`, `packages/stats/*`, `packages/sdk/js`, `packages/slack`로 구성됩니다. 빌드는 Turbo와 Bun 스크립트에 의존합니다.

## 4. 실행 흐름

### 텍스트 흐름도
```text
CLI entry ([packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1))
  -> yargs command dispatch
  -> bootstrap / instance runtime
  -> config / plugin / provider / catalog load
  -> local mode or attach mode 선택
  -> session create / continue / fork / share
  -> prompt admit (durable session_input)
  -> SessionRunCoordinator wake/run
  -> SessionRunner.run
  -> SessionContextEpoch initialize/prepare
  -> history load + system context render
  -> LLM.request + llm.stream
  -> tool-call publish
  -> ToolRegistry.settle
  -> tool result publish
  -> next turn or idle
  -> TUI / stdout render
  -> 종료
```

### 실제 흐름
- 시작점은 [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1)입니다.
- yargs가 `run`, `serve`, `web`, `session`, `plugin`, `mcp`, `attach`, `db` 등 명령을 분기합니다.
- `run`은 [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L1)에서 비대화형/대화형/attach 모드를 나눕니다.
- 대화형 UI는 [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L221)에서 provider stack을 세팅하고 렌더링합니다.
- 세션 실행은 [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L82)로 내려가며, 여기서 history/context/model/tool-loop가 수행됩니다.

### 중요한 분기
- 비대화형 모드에서는 permission rule 일부를 기본 deny로 설정합니다. `question`, `plan_enter`, `plan_exit`가 deny됩니다.
  근거: [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L365)
- `--dangerously-skip-permissions`는 “explicit deny가 아닌 것은 자동 승인” 성격이지만, 명시적 위험 옵션입니다.
  근거: [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L231)
- attach 모드에서는 로컬 인스턴스 대신 서버에 붙습니다.
- interactive local 모드는 in-process server를 사용합니다.

## 5. CLI/TUI 구조

### TUI 프레임워크
- `@opentui/core` 계열의 renderable을 쓰고, `@opentui/solid` + SolidJS로 UI를 구성합니다.
- 루트는 [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L228)이며, `ErrorBoundary` 아래에 `RouteProvider`, `SDKProvider`, `ProjectProvider`, `SyncProvider`, `ThemeProvider`, `DialogProvider`, `PromptHistoryProvider` 등을 겹겹이 얹습니다.
- 렌더러는 `createCliRenderer` 기반입니다.
  근거: [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L131)

### 화면 구성
- 홈 화면: [packages/opencode/src/cli/cmd/tui/routes/home.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/home.tsx#L22)
- 세션 화면: [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L183)
- sidebar: [packages/opencode/src/cli/cmd/tui/routes/session/sidebar.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/sidebar.tsx#L12)
- footer: [packages/opencode/src/cli/cmd/tui/routes/session/footer.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/footer.tsx#L9)

### 입력창
- 입력은 `TextareaRenderable` 기반입니다.
  근거: [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L1)
- `onContentChange`, `onPaste`, `onSubmit`에서 prompt state를 갱신합니다.
- IME 대응을 위해 submit을 double-defer 합니다.
  근거: [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L930)
- 파일/이미지/PDF paste는 별도 attachment part로 변환됩니다.
  근거: [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L1234)

### 메시지 렌더링
- 세션 메시지 렌더링의 핵심은 [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1)입니다.
- `PART_MAPPING`을 통해 `text`, `reasoning`, `tool`, `compaction`, `write`, `edit`, `apply_patch`, `bash`, `webfetch`, `websearch`, `task`, `todowrite`, `question` 등을 개별 렌더러로 나눕니다.
- `TextPartDisplay`는 streaming 중 `PacedMarkdown`으로 점진 출력하고, 완료 후엔 `Markdown`으로 렌더링합니다.
  근거: [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1468)
- `ReasoningPartDisplay`도 같은 방식으로 markdown 렌더링합니다.
- `ToolPartDisplay`는 tool별 UI를 동적으로 고릅니다.
  근거: [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1359)

### Markdown 렌더링
- `marked`로 파싱하고, `DOMPurify`로 sanitize하며, `morphdom`으로 DOM diff 업데이트를 합니다.
  근거: [packages/ui/src/components/markdown.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L241)
- streaming 마크다운은 [packages/ui/src/components/markdown-stream.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown-stream.ts#L29)에서 블록 단위로 쪼개 처리합니다.
- 코드블록에는 copy button이 붙고, external link는 안전하게 정리됩니다.
  근거: [packages/ui/src/components/markdown.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L122)

### Diff 렌더링
- diff 표준화는 [packages/ui/src/components/session-diff.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/session-diff.ts#L1)에서 처리합니다.
- patch text가 전체 diff인지 부분 diff인지 판별하고, `@pierre/diffs`로 `FileDiffMetadata`를 만듭니다.
- 세션 turn diff는 [packages/ui/src/components/session-turn.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/session-turn.tsx#L241)에서, review diff는 [packages/ui/src/components/session-review.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/session-review.tsx#L164)에서 렌더링합니다.
- 별도 diff viewer는 [packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx#L78)에서 git diff / last-turn diff를 전환합니다.

### Spinner / status
- spinner는 [packages/opencode/src/cli/cmd/tui/component/spinner.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/spinner.tsx#L10)에서 `opentui-spinner/solid`를 사용합니다.
- `animations_enabled` kv 값이 false면 정적 텍스트 fallback으로 바뀝니다.

### Slash command / autocomplete
- slash command와 mention autocomplete는 [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L73)에서 구현됩니다.
- `@` 트리거와 `/` 트리거를 모두 처리합니다.
- 파일, reference, agent, MCP resource를 후보로 넣고 `fuzzysort`로 정렬합니다.
  근거: [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L332), [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L476), [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L507)

### 키보드/리사이즈/테마
- 키맵은 [packages/opencode/src/cli/cmd/tui/keymap.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/keymap.tsx#L1)에서 관리합니다.
- `leader`, command palette, input binding, diff viewer shortcut을 모두 keymap 레이어가 담당합니다.
- `useTerminalDimensions()`를 사용해 폭/높이에 반응하는 레이아웃을 구성합니다.
  근거: [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L233)
- theme는 [packages/opencode/src/cli/cmd/tui/context/theme.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/context/theme.tsx#L552) 계열에서 markdown/diff 색상 토큰을 정의합니다.

### 에러 메시지
- 루트는 `ErrorBoundary`로 감싸고, `ErrorComponent`를 사용합니다.
- tool 오류는 `ToolErrorCard`, `DialogMessage`, permission/question prompt로 분기됩니다.

## 6. Agent Loop 구조

### 핵심 요약
agent loop의 중심은 [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L20)입니다. 이 파일의 주석 자체가 “한 provider turn마다 정확히 한 번 `llm.stream(request)`를 호출하고, tool execution과 continuation은 여기서 처리하라”고 못 박고 있습니다.

### pseudocode
```text
SessionRunner.run(sessionID)
  -> pending steer/queue 확인
  -> interrupted tool 실패 마킹
  -> while openActivity:
       for step in 1..25:
         session = load
         systemContext = SessionContextEpoch.initialize/prepare
         history = store.runnerContext(sessionID, baselineSeq)
         model = SessionRunnerModel.resolve(session)
         request = LLM.request({
           system: baseline,
           messages: toLLMMessages(history, model),
           tools: ToolRegistry.definitions()
         })
         stream = llm.stream(request)
         for each LLM event:
           publish event to durable session events
           if tool-call and providerExecuted != true:
             tool settlement = ToolRegistry.settle(call)
             publish toolResult back to stream
         await tool fibers
         handle interruptions / provider failure / question rejection
         if no continuation and no pending steer: break
       if still needs continuation: step limit error
       if queue pending: continue next queue activity
```

### 실제 구성 요소
- 세션 입력은 [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L18)에서 `steer` / `queue`로 나뉩니다.
- `admit()`가 durable `session_input` row를 만들고, `promoteSteers()` / `promoteNextQueued()`가 safe boundary에서 visible prompt로 올립니다.
- 세션 컨텍스트는 [packages/core/src/session/context.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context.ts#L65)에서 compaction 이후 메시지를 로드합니다.
- system context baseline/snapshot은 [packages/core/src/session/context-epoch.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context-epoch.ts#L35)에서 관리됩니다.
- tool call event publish와 결과 반영은 [packages/core/src/session/runner/publish-llm-event.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/publish-llm-event.ts#L59)에서 처리됩니다.
- durable history를 다시 LLM 메시지로 바꾸는 함수는 [packages/core/src/session/runner/to-llm-message.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/to-llm-message.ts#L139)입니다.

### 종료 조건 / 예외
- step 상한은 25입니다.
  근거: [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L80)
- question rejection은 tool loop를 중단시킵니다.
- provider가 실패하거나 interrupt가 걸리면 unsettled tool을 실패 처리합니다.
- provider retry 전략은 아직 완성형으로 보이지 않으며, 일부 TODO가 남아 있습니다. 상세 정책은 `확인 필요`입니다.

## 7. Provider / Model Adapter 구조

| Provider 관련 파일 | 역할 |
|---|---|
| [packages/core/src/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/provider.ts#L1) | provider ID, enable 방식(env/account/custom), API/request envelope 정의 |
| [packages/core/src/model.ts](D:/AREA51/workspace/opencode/packages/core/src/model.ts#L1) | model metadata, capability, cost, variant, parse helper |
| [packages/core/src/catalog.ts](D:/AREA51/workspace/opencode/packages/core/src/catalog.ts#L1) | provider/model registry, default 선택, availability, policy filtering |
| [packages/core/src/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/plugin/provider.ts#L1) | provider plugin 목록 등록 |
| [packages/core/src/plugin/boot.ts](D:/AREA51/workspace/opencode/packages/core/src/plugin/boot.ts#L54) | provider/plugin bootstrapping |
| [packages/core/src/session/runner/model.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/model.ts#L77) | 세션 실행 시 모델을 `@opencode-ai/llm` route로 변환 |
| [packages/llm/src/llm.ts](D:/AREA51/workspace/opencode/packages/llm/src/llm.ts#L30) | request normalization, generateObject, stream/generate 엔트리 |
| [packages/llm/src/tool.ts](D:/AREA51/workspace/opencode/packages/llm/src/tool.ts#L99) | tool schema/definition abstraction |
| [packages/llm/src/schema/events.ts](D:/AREA51/workspace/opencode/packages/llm/src/schema/events.ts#L76) | 표준 LLM event schema |

### 중요한 관찰
- provider catalog는 매우 넓습니다. `ProviderPlugins`에 Anthropic, OpenAI, Google, Bedrock, OpenRouter 등 다수가 등록됩니다.
  근거: [packages/core/src/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/plugin/provider.ts#L34)
- 하지만 실제 세션 러너가 `@opencode-ai/llm` route로 내리는 실행 경로는 현재 **openai / anthropic / openai-compatible**로 제한되어 있습니다.
  근거: [packages/core/src/session/runner/model.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/model.ts#L77)
- 즉, **catalog에 등록되는 provider 범위**와 **실행 가능한 model adapter 범위**가 완전히 같지는 않습니다. 이 차이는 구조적으로 중요합니다.
- token usage / cost tracking은 `LLM.Usage`, `ModelV2.Cost`, session `tokens_*`, `cost` 컬럼으로 이어집니다.
  근거: [packages/llm/src/schema/events.ts](D:/AREA51/workspace/opencode/packages/llm/src/schema/events.ts#L49), [packages/core/src/model.ts](D:/AREA51/workspace/opencode/packages/core/src/model.ts#L23), [packages/core/src/session/sql.ts](D:/AREA51/workspace/opencode/packages/core/src/session/sql.ts#L41)

## 8. Tool System 구조

### 구조
- core tool registry: [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L1)
- built-in tools list: [packages/core/src/tool/builtins.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/builtins.ts#L30)
- application tools: [packages/core/src/tool/application-tools.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/application-tools.ts#L25)

### 동작 원리
- `ToolRegistry.definitions()`는 local tool과 application tool을 합쳐 LLM에 노출할 tool definition을 만듭니다.
- `ToolRegistry.settle()`는 schema decode -> authorize -> execute -> encode 순서로 tool call을 정산합니다.
- invalid input은 `ToolFailure`로 변환됩니다.
  근거: [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L135)

### built-in tool 표

| Tool | 역할 | 입력 | 출력 | 승인/권한 필요 여부 |
| --- | --- | --- | --- | --- |
| `read` | 파일/디렉터리/managed output 읽기 | `path`/`reference` 또는 `resource` | content/page/list | 예, `read` |
| `write` | 단일 파일 쓰기 | `path`, `content` | write result | 예, `edit`, 외부면 `external_directory` 추가 |
| `edit` | 정확한 문자열 치환 | `path`, `oldString`, `newString`, `replaceAll?` | diff-style success | 예, `edit`, 외부면 `external_directory` 추가 |
| `apply_patch` | add/update/delete 패치 적용 | `patchText` | applied 파일 목록 | 예, `edit`, 외부면 `external_directory` 추가 |
| `bash` | 쉘 명령 실행 | `command`, `workdir?`, `timeout?`, `description?` | stdout/stderr preview, exit code, truncation info | 예, `bash`, 외부 workdir면 `external_directory` 추가 |
| `grep` | 파일 내용 regex 검색 | `pattern`, `path?`, `reference?`, `include?`, `limit?` | match list | 예, `grep` |
| `glob` | 파일/디렉터리 glob 검색 | `pattern`, `path?`, `reference?`, `limit?` | path list | 예, `glob` |
| `question` | 사용자에게 질문 | structured question | answer/decision | 사용자 응답 필요 |
| `skill` | skill body 로드 | skill ref/name | skill content | 예, `skill` |
| `todowrite` | session todo 갱신 | todo array/state | updated todo list | 예, `todowrite` |
| `webfetch` | URL fetch | `url` | fetched content | 예, `webfetch` |
| `websearch` | 검색 쿼리 실행 | `query` | search results | 예, `websearch` |

### 중요한 보안 포인트
- `bash`는 호스트 사용자 filesystem/process/network authority로 실행됩니다.
  근거: [packages/core/src/tool/bash.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/bash.ts#L75)
- `bash`의 “외부 경로 탐지”는 advisory only입니다. parser 기반 강제 샌드박스가 아닙니다.
- `apply_patch`는 순차 적용이며, 나중 단계 실패 시 앞선 변경이 되돌려지지 않습니다.
  근거: [packages/core/src/tool/apply-patch.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/apply-patch.ts#L36)
- `read` / `write` / `edit` / `apply_patch`는 모두 `LocationMutation`과 `FileMutation`을 통해 경계와 재검증을 겁니다.
- `ToolOutputStore`는 긴 출력물을 `tool-output://...` managed resource로 분리합니다.
  근거: [packages/core/src/tool-output-store.ts](D:/AREA51/workspace/opencode/packages/core/src/tool-output-store.ts#L17)

### 추가 관찰
- [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1)는 tool별 렌더러를 따로 두어서, tool execution과 UI 표시가 분리되어 있습니다.
- `task` tool은 UI 렌더러는 확인됐지만 core tool 구현 위치는 이번 읽기 범위에서 확인되지 않았습니다. `확인 필요`입니다.

## 9. Context / Memory / Session 관리

### 대화 history 저장
- 세션 메시지는 [packages/core/src/session/sql.ts](D:/AREA51/workspace/opencode/packages/core/src/session/sql.ts#L117)의 `session_message` 테이블에 저장됩니다.
- message payload는 [packages/core/src/session/message.ts](D:/AREA51/workspace/opencode/packages/core/src/session/message.ts#L179)의 tagged union입니다.
- `user`, `assistant`, `system`, `shell`, `synthetic`, `compaction`, `agent-switched`, `model-switched`가 명시되어 있습니다.

### session 재개 구조
- `SessionContext.load()`는 최신 compaction과 baseline_seq 이후 메시지를 로드합니다.
  근거: [packages/core/src/session/context.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context.ts#L65)
- `toLLMMessages()`는 durable history를 provider-facing message 배열로 다시 낮춥니다.
  근거: [packages/core/src/session/runner/to-llm-message.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/to-llm-message.ts#L139)

### prompt admission / promotion
- `SessionInput`은 `steer`와 `queue`를 분리합니다.
  - `steer`: active provider turn boundary에 steering prompt로 반영
  - `queue`: 다음 activity로 FIFO 처리
- `admit()`와 `promoteSteers()`/`promoteNextQueued()`는 durable row를 통해 safe boundary를 보장합니다.
  근거: [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L54)

### system context / memory
- `SystemContext`는 단순 문자열이 아니라, **독립적으로 재관측 가능한 typed source 집합**입니다.
  근거: [packages/core/src/system-context.ts](D:/AREA51/workspace/opencode/packages/core/src/system-context.ts#L31)
- baseline/snapshot/reconciliation/replacement이 구현되어 있어, context가 변하면 update text를 생성하거나 replacement를 대기합니다.
- ambient instructions는 [packages/core/src/instruction-context.ts](D:/AREA51/workspace/opencode/packages/core/src/instruction-context.ts#L21)에서 `AGENTS.md`를 읽습니다.
- 글로벌 `AGENTS.md`와 프로젝트 상향 `AGENTS.md`를 합칩니다.
- `Config.Info.instructions` 필드도 존재하지만, **config.instructions가 최종 system context에 어떻게 완전히 합성되는지는 이번 읽기 범위에서 완전 추적되지 않았습니다. 확인 필요**입니다.
  근거: [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L92), [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L40)

### shared state
- session-level 공유 상태는 DB와 event bus 중심입니다.
- global memory처럼 모든 세션이 공유하는 단일 메모리는 보이지 않았고, saved permissions / global config / plugin state가 각기 다른 범위로 나뉩니다.

## 10. File / Diff / Patch 처리

### 읽기
- 파일 읽기는 [packages/core/src/filesystem.ts](D:/AREA51/workspace/opencode/packages/core/src/filesystem.ts#L17)에서 canonical root와 containment를 검사합니다.
- `.gitignore`와 `.ignore`를 읽어 ignore 패턴을 적용합니다.
  근거: [packages/core/src/filesystem.ts](D:/AREA51/workspace/opencode/packages/core/src/filesystem.ts#L187)
- 프로젝트 reference는 read-oriented이며, mutation tool에서는 받아들이지 않는 방향입니다.
  근거: [packages/core/src/tool/write.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/write.ts#L1)

### 쓰기 / 수정 / patch
- `LocationMutation.resolve()`가 relative path escape, symlink, external directory boundary를 잡습니다.
  근거: [packages/core/src/location-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/location-mutation.ts#L11)
- `FileMutation`은 canonical target 기준 keyed mutex를 걸고, revalidate 후 즉시 mutate합니다.
  근거: [packages/core/src/file-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/file-mutation.ts#L76)
- `edit`는 exact replace를 수행하고, `writeIfUnchanged`로 stale content를 막습니다.
  근거: [packages/core/src/tool/edit.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/edit.ts#L123)
- `apply_patch`는 여러 파일을 순차 적용하며, atomic rollback은 없습니다.
  근거: [packages/core/src/tool/apply-patch.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/apply-patch.ts#L36)

### diff 생성 / 렌더링
- `session-diff.ts`가 patch/before-after/VCS diff를 `FileDiffMetadata`로 정규화합니다.
- `apply-patch-file.ts`는 raw patch object를 UI가 쓰는 구조로 바꿉니다.
- `session-review.tsx`와 `diff-viewer.tsx`는 `@pierre/diffs` 기반의 split/unified view를 제공합니다.

### user approval 후 write
- write/edit/apply_patch/bash/read/glob/grep/webfetch/websearch는 승인 레이어를 거칩니다.
- `FileMutation.revalidate()`는 승인 사이에 target이 바뀌었는지 다시 체크합니다.
  근거: [packages/core/src/location-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/location-mutation.ts#L279)

### rollback / undo
- session-level revert/unrevert API는 존재합니다.
  근거: [packages/opencode/src/server/routes/instance/httpapi/groups/session.ts](D:/AREA51/workspace/opencode/packages/opencode/src/server/routes/instance/httpapi/groups/session.ts#L369)
- TUI에도 `session.undo`/`session.redo` 키바인딩과 revert diff preview가 있습니다.
  근거: [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L1126)
- 다만 low-level 파일 mutation 자체는 atomic rollback을 보장하지 않습니다.

## 11. Permission / Security 구조

### 권한 엔진
- 권한 엔진은 [packages/core/src/permission.ts](D:/AREA51/workspace/opencode/packages/core/src/permission.ts#L101)에서 구현됩니다.
- ruleset은 마지막 매칭 rule이 우선입니다.
- action/resource 단위로 allow/deny/ask가 평가됩니다.
- saved permissions은 project 범위로 남고, agent permissions와 합쳐집니다.

### approval 구조
- `PermissionV2.assert()`는 ask면 대기, deny면 즉시 오류, allow면 통과합니다.
- `reply("always")`는 saved permission으로 기록합니다.
  근거: [packages/core/src/permission.ts](D:/AREA51/workspace/opencode/packages/core/src/permission.ts#L239)

### path / workspace boundary
- relative path는 location 밖으로 escape할 수 없습니다.
- absolute external path는 별도 `external_directory` 승인 없이는 안 됩니다.
- canonical target과 filesystem identity(dev/ino)가 재검증됩니다.
  근거: [packages/core/src/location-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/location-mutation.ts#L253), [packages/core/src/file-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/file-mutation.ts#L106)

### network / shell / secret
- `bash`는 명시적으로 host authority를 사용합니다. 별도 샌드박스는 확인되지 않았습니다.
- `webfetch` / `websearch`는 네트워크 접근 도구입니다.
- secret redaction 전용 계층은 이번 읽기 범위에서 확인되지 않았습니다. `확인 필요`입니다.
- command injection 방지는 parser-based hard sandbox라기보다, 승인 + path boundary + output truncation 중심입니다.

### audit trail
- permission ask/reply, session events, tool events가 이벤트 로그 역할을 합니다.
- 별도의 “보안 감사 로그” 전용 스토어는 확인되지 않았습니다.

## 12. Configuration 구조

### 로딩 위치
- 글로벌 config와 project config를 모두 읽습니다.
  근거: [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L125)
- 대상 파일은 `config.json`, `opencode.json`, `opencode.jsonc`입니다.
- `.opencode` 디렉터리도 상위 탐색 대상입니다.
- `OPENCODE_DISABLE_PROJECT_CONFIG`가 켜지면 project-side ambient instruction 로딩이 막힙니다.
  근거: [packages/core/src/instruction-context.ts](D:/AREA51/workspace/opencode/packages/core/src/instruction-context.ts#L46)

### 설정 항목
- `shell`
- `model`
- `autoupdate`
- `share`
- `enterprise`
- `username`
- `permissions`
- `agents`
- `snapshots`
- `watcher`
- `formatter`
- `lsp`
- `attachments`
- `tool_output`
- `mcp`
- `compaction`
- `skills`
- `commands`
- `instructions`
- `references`
- `plugins`
- `experimental`
- `providers`

### 우선순위
- 일반 설정은 global -> project file -> `.opencode` 순으로 더 구체적인 값이 우선됩니다.
- 정책 rule은 반대로 뒤집어 적용해서 user-global rule이 repo rule을 덮을 수 있게 합니다.
  근거: [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L199)

### TUI 전용 설정
- keybind, theme mode, prompt height, diff style, animations 등은 별도 TUI config로 분리되어 있습니다.
- 관련 파일은 [packages/opencode/src/cli/cmd/tui/config/tui.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/config/tui.ts)와 [packages/opencode/src/cli/cmd/tui/config/keybind.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/config/keybind.ts)입니다.

## 13. Build / Packaging / Distribution

### 요약
- package manager: **Bun**
  근거: [package.json](D:/AREA51/workspace/opencode/package.json#L1)
- monorepo: **yes**, Bun workspaces + Turbo
- 주요 앱/패키지:
  - CLI: `packages/opencode`
  - core: `packages/core`
  - TUI shared UI: `packages/ui`
  - web app: `packages/app`
  - desktop: `packages/desktop`
  - server: `packages/server`
  - JS SDK: `packages/sdk/js`

### 실행/개발 스크립트
- `bun run --cwd packages/opencode --conditions=browser src/index.ts`
- `bun --cwd packages/desktop dev`
- `bun --cwd packages/app dev`
- `bun run --cwd packages/console/app dev`
- `bun turbo typecheck`
- `bun run script/upgrade-opentui.ts`
- `bun run --cwd packages/core fix-node-pty`
  근거: [package.json](D:/AREA51/workspace/opencode/package.json#L7)

### 배포 관찰
- desktop은 Electron 계열 의존성이 있고, web은 Vite/Solid 기반입니다.
- server는 Hono 기반 API를 사용합니다.
- SDK는 별도 build script로 재생성하는 구조입니다. AGENTS 지침상 `./packages/sdk/js/script/build.ts`를 사용합니다.
- root test는 의도적으로 실패하게 막혀 있습니다.
- Windows/macOS/Linux 대응은 일부 구현이 보이지만, 실제 전 범위 호환성은 확인 필요입니다.

### 남는 리스크
- provider/server/plugin 별 의존성이 많아서 bootstrap 비용이 큽니다.
- `node-pty`, Electron, MCP server 등 외부 의존성이 많아 install/update 안정성은 환경 영향을 받습니다.
- offline 환경에서 완전 동작 가능성은 확인되지 않았습니다.

## 14. 주요 설계 패턴

| 설계 패턴 | 구현 위치 | 설명 | 장점 | 단점 |
|---|---|---|---|---|
| location-scoped DI graph | [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L1) | 세션/툴/파일/권한/모델을 location 범위로 묶음 | 경계가 명확함 | 초기 조립 복잡 |
| system context source/snapshot | [packages/core/src/system-context.ts](D:/AREA51/workspace/opencode/packages/core/src/system-context.ts#L31) | context를 재관측 가능한 typed source로 관리 | 재개/갱신 용이 | 추상화 비용 큼 |
| session context epoch | [packages/core/src/session/context-epoch.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context-epoch.ts#L35) | baseline/snapshot/revision으로 durable context 관리 | 변화 감지 정확 | 이해 난이도 높음 |
| prompt admit/promote | [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L54) | prompt를 durable row로 먼저 기록한 뒤 safe boundary에서 승격 | 재개/steer/queue가 명확 | 개념이 낯설 수 있음 |
| tool registry + settlement | [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L108) | schema/authorize/execute/encode를 한 레이어에 집중 | tool 확장 쉬움 | 권한 흐름 추적 필요 |
| provider/model adapter | [packages/core/src/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/provider.ts#L47), [packages/core/src/session/runner/model.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/model.ts#L77) | model metadata와 실행 route를 분리 | provider 다양성 확보 | 실행 지원 범위가 따로 노출될 수 있음 |
| markdown streaming renderer | [packages/ui/src/components/markdown.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L241), [packages/ui/src/components/markdown-stream.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown-stream.ts#L29) | streaming markdown을 sanitize + morphdom으로 갱신 | UX 좋음 | DOM 복잡도 증가 |
| diff normalization | [packages/ui/src/components/session-diff.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/session-diff.ts#L1) | patch/before-after/VCS diff를 통일 | 렌더러 재사용 | patch 해석 로직이 복잡 |
| TUI plugin slots | [packages/opencode/src/cli/cmd/tui/plugin/runtime.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/plugin/runtime.ts#L1) | screen slot을 plugin이 replace/single_winner로 채움 | 확장성 좋음 | 런타임 이해 비용 큼 |
| layered config loader | [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L125) | global/project/.opencode config를 합성 | 사용자/프로젝트 분리 | 우선순위 해석 복잡 |

## 15. 장점

- **사용자 경험**
  - 홈/세션/diff/review가 분리되어 있고, 입력창과 타임라인이 자연스럽게 연결됩니다.
- **터미널 UI**
  - OpenTUI + SolidJS + reactive dimensions 조합으로, 텍스트 UI지만 꽤 풍부한 상호작용을 제공합니다.
- **agent loop**
  - session/admit/prompt/tool/result의 경계가 비교적 명확합니다.
- **tool system**
  - schema 기반 tool registry와 permission approval이 분리되어 있어 확장과 검증이 쉽습니다.
- **provider abstraction**
  - catalog/model/provider adapter가 분리되어 provider 추가 경로가 있습니다.
- **context/session 관리**
  - system context epoch와 durable session history가 있어 resume/replay에 강합니다.
- **보안/권한**
  - relative path containment, external_directory 승인, revalidation이 잘 들어가 있습니다.
- **확장성**
  - TUI plugin slots, application tools, provider plugins, config plugins가 모두 존재합니다.
- **유지보수성**
  - core / ui / opencode CLI / server / desktop / sdk가 패키지 단위로 나뉘어 있습니다.
- **배포 편의성**
  - Bun workspace + Turbo로 개발/빌드/타입체크가 통합되어 있습니다.

## 16. 단점 및 리스크

| 리스크 | 설명 | 영향도 | 확인한 근거 |
|---|---|---:|---|
| 구조 복잡도 | CLI, TUI, core, UI, server, desktop, SDK가 얽혀 있어 학습 곡선이 높음 | 높음 | [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L1), [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L228) |
| shell 안전성 | `bash`는 호스트 authority로 실행되고 hard sandbox가 아님 | 높음 | [packages/core/src/tool/bash.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/bash.ts#L75) |
| atomic rollback 부재 | `apply_patch`는 순차 적용이며 partial apply 가능 | 높음 | [packages/core/src/tool/apply-patch.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/apply-patch.ts#L36) |
| provider adapter 범위 제한 | catalog에 비해 실제 session runner가 지원하는 route는 제한적 | 중간 | [packages/core/src/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/plugin/provider.ts#L34), [packages/core/src/session/runner/model.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/model.ts#L77) |
| 아직 남은 TODO | provider retry, compaction, cancellation, undo 등 여러 축이 TODO로 남아 있음 | 중간 | [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L60), [packages/core/src/file-mutation.ts](D:/AREA51/workspace/opencode/packages/core/src/file-mutation.ts#L92) |
| 운영 범위 큼 | plugin, MCP, server, desktop, web, CLI 모두 있어서 유지보수 범위가 큼 | 중간 | 루트 workspace와 다수 package 구조 |
| 문서/테스트 확인 부족 | 이번 읽기 범위에서 모든 패키지 테스트와 릴리즈 자동화를 끝까지 추적하지 못함 | 중간 | `확인 필요` |

## 17. 다른 도구와 비교

| 도구 | opencode와 유사점 | 차이점 |
|---|---|---|
| Claude Code | 터미널 중심 agent UX, 파일 수정/쉘 실행/승인 흐름이 유사 | opencode는 더 플랫폼형이고, TUI/서버/desktop/web/sdk까지 포함한 monorepo 구조가 더 강함 |
| OpenAI Codex CLI | CLI에서 프롬프트를 보내고 tool을 쓰는 agent 경험이 유사 | opencode는 durable session, plugin slots, TUI review/diff가 더 전면에 있음 |
| Aider | 코드 수정과 diff 검토를 중심으로 쓰는 점이 유사 | opencode는 git-first 보조도구보다 agent runtime/platform 성격이 강함 |
| Cline | 승인 기반 tool use와 interactive coding assistant라는 점이 유사 | Cline은 IDE 확장 중심, opencode는 terminal/TUI 중심 |
| OpenHands | 더 자율적인 agent platform이라는 점과 다중 tool orchestration이 유사 | opencode는 local terminal + durable session + TUI가 핵심이고, 사용자 제어와 로컬 상호작용이 더 강함 |

## 18. 핵심 소스 파일 상세 분석

| 파일 | 왜 중요한가 | 핵심 함수/클래스 | 요약 |
|---|---|---|---|
| [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1) | 모든 CLI 명령의 시작점 | `yargs`, middleware, `.command(...)` | process metadata, logging, command dispatch를 담당 |
| [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L122) | 실제 사용자 실행 경로의 중심 | `RunCommand`, `session()`, `attachSDK()`, `submit` 관련 흐름 | non-interactive, interactive, attach를 모두 처리 |
| [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L221) | TUI 전체 조립 | `mountTui`, `tuiRendererConfig`, provider stack | route/theme/sdk/sync/dialog/plugin을 한 화면에 연결 |
| [packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx#L911) | 입력/submit/첨부의 핵심 | `submit()`, `submitInner()`, `pasteAttachment()` | session.create/prompt/command/shell 분기 |
| [packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/prompt/autocomplete.tsx#L73) | 입력 보조 기능의 중심 | `Autocomplete`, `Reference.resolveAll`, `fuzzysort` | slash command, @mention, files, agents, MCP 후보 제공 |
| [packages/opencode/src/cli/cmd/tui/routes/session/index.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L183) | 세션 화면의 실질적인 오케스트레이터 | `Session()`, `AssistantMessage`, `ReasoningPart`, `TextPart`, `ToolPart` | 타임라인, prompt, permissions, questions, sidebar를 조립 |
| [packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/feature-plugins/system/diff-viewer.tsx#L78) | diff review 전용 UI | `DiffViewer`, `normalizeDiffs`, file tree helpers | git diff / last-turn diff를 다루는 별도 viewer |
| [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L46) | runtime의 큰 그림 | `LocationServiceMap.lookup` | location-scoped service graph를 실제로 조합 |
| [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L82) | agent loop 본체 | `runTurn`, `run`, `awaitToolFibers` | prompt/history/model/tool/continuation을 반복 |
| [packages/core/src/session/runner/publish-llm-event.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/publish-llm-event.ts#L59) | provider stream을 durable session event로 변환 | `createLLMEventPublisher` | text/reasoning/tool call/result를 세션 이벤트로 기록 |
| [packages/core/src/session/runner/to-llm-message.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/to-llm-message.ts#L94) | durable history를 모델 메시지로 복원 | `toLLMMessages` | user/system/shell/assistant/compaction message를 변환 |
| [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L54) | durable prompt admission/promotion | `admit`, `promoteSteers`, `promoteNextQueued` | steer/queue semantics의 핵심 |
| [packages/core/src/session/context.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context.ts#L65) | 세션 history loading | `load`, `loadForRunner` | compaction과 baseline 이후 메시지 선택 |
| [packages/core/src/session/context-epoch.ts](D:/AREA51/workspace/opencode/packages/core/src/session/context-epoch.ts#L35) | system context의 durable baseline | `initialize`, `prepare`, `requestReplacement` | context 갱신/대체를 durable하게 추적 |
| [packages/core/src/system-context.ts](D:/AREA51/workspace/opencode/packages/core/src/system-context.ts#L130) | system context 추상화 | `make`, `combine`, `initialize`, `reconcile`, `replace` | baseline/update/remove 렌더링과 snapshot 관리 |
| [packages/core/src/instruction-context.ts](D:/AREA51/workspace/opencode/packages/core/src/instruction-context.ts#L21) | ambient instruction loader | `observe`, `render` | `AGENTS.md`를 system context로 합침 |
| [packages/core/src/config.ts](D:/AREA51/workspace/opencode/packages/core/src/config.ts#L125) | config loader | `loadFile`, `loadDirectory`, `entries` | global/project/.opencode 설정 우선순위 처리 |
| [packages/core/src/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/provider.ts#L47) | provider schema | `ProviderV2.Info` | env/account/custom enable 및 API envelope 정의 |
| [packages/core/src/model.ts](D:/AREA51/workspace/opencode/packages/core/src/model.ts#L55) | model schema | `ModelV2.Info`, `parse` | capability/cost/limit/variant까지 포함한 메타데이터 |
| [packages/core/src/catalog.ts](D:/AREA51/workspace/opencode/packages/core/src/catalog.ts#L89) | provider/model catalog | `provider.all`, `model.default`, `model.small` | plugin-driven registry와 policy filtering |
| [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L108) | tool settlement core | `definitions`, `settle`, `execute` | tool schema 검증, 승인, 실행, 결과 프로젝트 |
| [packages/core/src/tool/bash.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/bash.ts#L75) | 위험도가 높은 tool | `execute` | shell command 실행, timeout, output truncation |
| [packages/core/src/tool/edit.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/edit.ts#L81) | exact edit | `writeIfUnchanged`, BOM/line-ending 처리 | 정확한 치환과 stale check를 수행 |
| [packages/core/src/tool/apply-patch.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/apply-patch.ts#L64) | multi-file patch | `planned`, `prepared`, sequential apply | partial apply 가능성을 명시적으로 남김 |
| [packages/ui/src/components/markdown.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/markdown.tsx#L241) | markdown renderer | `Markdown`, `sanitize`, `setupCodeCopy` | streaming markdown을 안전하게 렌더링 |
| [packages/ui/src/components/message-part.tsx](D:/AREA51/workspace/opencode/packages/ui/src/components/message-part.tsx#L1359) | message/tool renderer | `PART_MAPPING` | assistant text/reasoning/tool UI를 개별화 |
| [packages/ui/src/components/session-diff.ts](D:/AREA51/workspace/opencode/packages/ui/src/components/session-diff.ts#L30) | diff 정규화 | `resolveFileDiff`, `normalize` | patch/before-after/VCS diff 통합 |
| [packages/opencode/src/cli/cmd/tui/plugin/runtime.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/plugin/runtime.ts#L47) | TUI plugin runtime | plugin slots/runtime support | home/session/sidebar 등에 plugin replacement 가능 |

## 19. 최종 결론

1. **opencode의 핵심 아키텍처는 무엇인가?**
   - location-scoped DI graph 위에 session runner, tool registry, provider/catalog, permission engine, TUI renderer를 쌓은 **레이어드 agent platform**입니다.
   - 핵심 파일은 [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L46)와 [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L82)입니다.

2. **opencode의 가장 중요한 실행 흐름은 무엇인가?**
   - `prompt admit -> session runner wake -> system context initialize/prepare -> history load -> llm.stream -> tool settle -> tool result publish -> next turn`입니다.
   - 이 흐름은 [packages/core/src/session/input.ts](D:/AREA51/workspace/opencode/packages/core/src/session/input.ts#L54), [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L130), [packages/core/src/session/runner/publish-llm-event.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/publish-llm-event.ts#L221)에서 확인됩니다.

3. **opencode에서 가장 잘 설계된 부분은 무엇인가?**
   - `session_input`의 admit/promote 분리, `SessionContextEpoch`의 baseline/snapshot/revision, `LocationMutation`의 boundary/revalidation, `ToolRegistry`의 schema/permission/execute 분리입니다.
   - 즉, **안전성과 재개 가능성**을 구조적으로 분리해 둔 점이 가장 좋습니다.

4. **opencode에서 가장 복잡하거나 이해하기 어려운 부분은 무엇인가?**
   - `SessionRunner` + `SystemContextEpoch` + `ToolRegistry` + `PermissionV2` + TUI plugin runtime이 맞물리는 부분입니다.
   - 특히 durable prompt admission, context replacement, tool settlement, stream publish가 같은 세션 루프 안에서 섞여 있어, 처음 읽으면 복잡합니다.

5. **opencode를 학습할 때 가장 먼저 읽어야 할 파일은 무엇인가?**
   - 1순위: [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1)
   - 2순위: [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L46)
   - 3순위: [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L82)
   - 4순위: [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L228)

6. **opencode를 기반으로 커스텀 CLI/Agent를 만들려면 어떤 부분을 먼저 이해해야 하는가?**
   - `location-layer`의 조립 방식
   - `session runner`의 turn/continue 규칙
   - `ToolRegistry`와 `PermissionV2`
   - `Config`와 `SystemContext`
   - TUI가 필요하면 `app.tsx`, `prompt/index.tsx`, `message-part.tsx`

7. **opencode를 그대로 fork해서 활용하기 쉬운 구조인가, 아니면 참고용으로 읽는 것이 더 적합한 구조인가?**
   - 둘 다 가능하지만, 성격상 **“참고용으로 읽는 가치가 매우 크고, 실사용 fork도 가능하나 범위가 넓다”**가 맞습니다.
   - 단순 CLI fork보다 훨씬 큰 플랫폼이라, 가벼운 커스터마이즈는 어렵고, **비슷한 agent platform을 만들려는 경우**에 가장 잘 맞습니다.
   - 반대로 “작은 터미널 에이전트”를 만들 목적이면 전체 fork보다 핵심 레이어만 참고하는 편이 낫습니다.

## 20. 후속 분석이 필요한 부분

이번 읽기에서 남아 있는 확인 필요 항목은 다음입니다.

- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`의 상세 렌더링 분기 일부
- `packages/opencode/src/server/routes/instance/httpapi/groups/session.ts`의 revert/unrevert end-to-end 동작
- provider별 세부 플러그인 구현 전체
- `task` tool의 core 구현 위치
- release/upgrade/install 배포 자동화 세부
- 테스트 coverage와 실제 실행 결과

원하면 다음 단계로는 아래 중 하나를 이어서 정리할 수 있습니다.

1. `session runner`만 더 깊게 파서 pseudo-code를 더 정교하게 재구성
2. `TUI`만 따로 파서 화면/단축키/렌더링 레이어를 더 자세히 정리
3. `tool system`만 따로 파서 권한/안전성/patch/edit/write 흐름을 도식화
