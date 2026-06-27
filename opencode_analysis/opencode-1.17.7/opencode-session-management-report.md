# opencode Session 관리 정책 분석

작성일: 2026-06-18

## 결론 요약

opencode의 session은 "프로세스 실행 1회"가 아니라 대화와 작업 이력을 묶는 영속 식별자다. TUI, 앱, CLI 프로세스가 종료되어도 session row와 message/part row는 DB에 남고, 같은 `sessionID`로 다시 조회하거나 이어갈 수 있다.

session ID는 `ses_` 접두사를 가진 문자열이다. 일반 생성 경로는 `SessionID.descending()` 또는 V2의 `SessionSchema.ID.create()`이고, HTTP API에서는 생성 응답으로 받은 ID를 이후 `/session/{sessionID}/...` 경로 파라미터로 계속 전달한다. HTTP 인증에는 별도의 Basic `Authorization` 헤더를 쓰며, session ID는 API 인증 토큰처럼 헤더에 넣는 구조가 아니다.

다만 모델 호출 단계에서는 session ID가 헤더와 cache key로 전달된다. legacy opencode 런타임은 opencode provider에는 `x-opencode-session`, 일반 provider에는 `x-session-affinity`와 `X-Session-Id`를 넣는다. provider transform에서도 OpenAI/Azure/OpenRouter/Venice 계열 prompt cache key로 session ID를 쓸 수 있다.

`/new`와 alias `/clear`는 기존 session을 삭제하거나 DB를 비우는 명령이 아니다. TUI에서는 home route로 이동하고 입력 다이얼로그를 비우는 동작만 한다. 앱도 session ID 없는 새 session 화면으로 이동한다. 새 session ID는 그 화면에서 다음 prompt를 제출할 때 `session.create`가 호출되면서 새로 발급된다.

## 기존 분석 자료와의 관계

`REPORT.md`의 기존 문서들은 system prompt, `/init`, agent별 prompt 교체, workflow 전달 계약을 중심으로 분석되어 있다. session 관리 관점에서 연결되는 핵심은 다음과 같다.

- system prompt 조립과 model request 준비는 특정 `sessionID`의 message history를 기준으로 수행된다.
- `/init`, `explore`, `summary`, `compaction`, subagent 실행은 session 또는 child 실행 단위를 만들 수 있지만, 사용자 UI에서 보는 기본 대화 단위는 `ses_...` session row다.
- adapter나 외부 provider에서 opencode 요청을 추적하려면 user prompt 하나만 보지 말고 `sessionID`, message ID, provider headers, prompt cache key를 함께 봐야 한다.

## ID 생성 정책

session ID schema는 core V2 session schema에 있다.

- `packages/core/src/session/schema.ts:7` 부근: session ID는 `"ses"`로 시작하는 문자열 schema다.
- `packages/core/src/session/schema.ts:15` 부근: `create()`는 `"ses_" + Identifier.descending()` 형태로 생성한다.
- `packages/core/src/session/schema.ts:18` 부근: `descending(id?)`는 id가 없으면 새로 만들고, id가 있으면 schema 검증 후 그대로 채택한다.
- `packages/core/src/session/schema.ts:19` 부근: `fromExternal(input)`은 외부 ID를 deterministic hash로 바꿔 `ses_...`를 만든다.
- `packages/core/src/util/identifier.ts:14`와 `packages/core/src/util/identifier.ts:28` 부근: `Identifier.descending()`은 시간/카운터와 랜덤 문자열을 섞어 정렬 가능한 ID 본문을 만든다.

legacy opencode session layer도 이 schema를 그대로 쓴다.

- `packages/opencode/src/session/schema.ts`: `SessionID = SessionV2.ID`로 core V2 ID schema를 재사용한다.
- `packages/opencode/src/session/session.ts:541` 부근: `createNext(...)`가 session 생성의 공통 내부 경로다.
- `packages/opencode/src/session/session.ts:555` 부근: `id: SessionID.descending(input.id)`로 ID를 만든다. caller가 id를 넘기지 않으면 새 ID가 발급되고, 넘기면 schema가 허용하는 ID를 채택한다.
- `packages/opencode/src/session/session.ts:577` 부근: 생성 후 `SessionV1.Event.Created`를 publish한다.

V2 core service에는 같은 의미의 생성 경로가 별도로 있다.

- `packages/core/src/session.ts:201` 부근: `input.id ?? SessionSchema.ID.create()`로 새 session ID를 정한다.
- 같은 함수는 이미 존재하는 ID면 기존 session을 반환한다. 즉 V2에서는 "같은 session ID 재사용은 기존 session 채택"이라는 정책이 명시되어 있다.

정리하면 일반 사용자가 새 대화를 시작할 때 발급되는 ID는 `ses_` 접두사와 descending identifier 본문을 가진다. 외부 시스템이 deterministic session mapping을 원하면 V2의 `fromExternal` 경로는 `ses_` + sha256 기반 64 hex 문자열을 만든다.

## 저장과 유지

session은 이벤트와 projector를 통해 DB row로 유지된다.

- `packages/core/src/session/sql.ts:21` 부근: `SessionTable`은 `id`를 primary key로 가지며 `project_id`, `workspace_id`, `parent_id`, `directory`, `path`, `title`, `agent`, `model`, `cost`, `tokens_*`, `time_*` 등을 저장한다.
- `packages/core/src/session/sql.ts:74` 이후: message, part, todo, V2 message/input/context epoch 테이블이 `session_id`로 `SessionTable.id`를 참조한다.
- `packages/core/src/session/sql.ts:139` 부근: `SessionInputTable`은 V2 durable prompt admission 상태를 `session_id`와 함께 저장한다.
- `packages/core/src/session/projector.ts:220` 부근: `SessionV1.Event.Created`를 받아 `SessionTable`에 insert한다.
- `packages/core/src/session/projector.ts:239` 이후: update, delete, message, part, prompt lifecycle 이벤트가 같은 `sessionID` 기준으로 projection된다.

따라서 opencode 프로세스가 종료되어도 session 자체가 자동 삭제되지는 않는다. 종료는 현재 TUI/worker/server 프로세스를 닫는 일이고, 대화 이력의 생명주기는 DB의 session row와 명시적 delete/archive/update API에 의해 결정된다.

## HTTP API에서의 사용 형태

사용자 클라이언트와 opencode 서버 사이에서는 session ID가 주로 URL path parameter다.

- `packages/opencode/src/server/routes/instance/httpapi/groups/session.ts:78` 부근: `SessionPaths`가 session API 경로를 정의한다.
- `POST /session`: 새 session 생성.
- `GET /session/:sessionID`, `DELETE /session/:sessionID`, `PATCH /session/:sessionID`: 조회/삭제/갱신.
- `POST /session/:sessionID/message`: prompt 전송.
- `POST /session/:sessionID/prompt_async`, `/command`, `/shell`, `/abort`, `/fork`, `/summarize`, `/revert` 등도 모두 path의 `:sessionID`를 기준으로 동작한다.

handler는 path에서 받은 ID를 service input에 다시 넣는다.

- `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`: `prompt` handler는 `ctx.params.sessionID`를 `promptSvc.prompt({ ...ctx.payload, sessionID })` 형태로 넘긴다.
- `command`, `shell`, `abort`, `fork`, `remove` 등도 같은 방식으로 path session을 사용한다.

SDK도 같은 경로 계약을 반영한다.

- `packages/sdk/js/src/v2/gen/sdk.gen.ts`: `session.create`는 `/session`, `session.prompt`는 `/session/{sessionID}/message`, 기타 session operation도 `/session/{sessionID}/...` 형태다.

반면 서버 접근 인증은 session ID와 별개다.

- `packages/opencode/src/server/auth.ts:47` 부근: `ServerAuth.headers()`는 Basic `Authorization` 헤더를 만든다.
- `packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts`: middleware는 `auth_token` query 또는 Basic auth credential을 검증한다.
- `packages/cli/src/services/daemon.ts:63`과 `packages/cli/src/services/daemon.ts:138` 부근: daemon client도 password 기반 `ServerAuth.headers(...)`를 사용한다.

즉 API 호출에서 session ID는 "어느 대화에 대한 operation인가"를 가리키는 resource identifier이고, 인증 토큰은 아니다.

## 모델 호출에서의 전달 형태

opencode가 provider/model로 request를 보낼 때는 session ID를 헤더와 provider option에 넣을 수 있다.

legacy session LLM request preparation:

- `packages/opencode/src/session/llm/request.ts:181` 부근: provider ID가 `opencode`로 시작하면 `x-opencode-session: input.sessionID`를 넣는다.
- `packages/opencode/src/session/llm/request.ts:187` 부근: 일반 provider 경로에서는 `x-session-affinity: input.sessionID`를 넣는다.
- `packages/opencode/src/session/llm/request.ts:188` 부근: 일반 provider 경로에서는 `X-Session-Id: input.sessionID`도 넣는다.
- `packages/opencode/src/session/llm/request.ts`의 같은 블록에서 `x-parent-session-id`도 optional로 전달한다.
- 이 파일의 `chat.headers` plugin hook 결과와 model headers가 최종 headers에 merge되므로 plugin/model 설정이 같은 key를 override할 수 있다.

provider option/cache:

- `packages/opencode/src/provider/transform.ts:1071`, `1101`, `1180`, `1187` 부근: provider별 조건에 따라 `promptCacheKey` 또는 `prompt_cache_key`로 `input.sessionID`를 넣는다.
- `packages/core/src/session/runner/llm.ts:218` 부근: V2 core runner도 OpenAI provider option의 `promptCacheKey`를 session ID 기반으로 구성한다.

관찰 포인트는 두 계층이 다르다는 점이다. HTTP API/TUI/app에서 session ID는 URL path resource ID이고, provider outbound request에서는 routing, affinity, cache, trace 목적의 header/option으로도 사용된다.

## prompt 실행과 session 소유권

prompt 실행은 session ID를 중심으로 message를 만들고, running state를 묶고, history를 다시 읽는다.

- `packages/opencode/src/session/prompt.ts`: `PromptInput`은 `sessionID`를 필수로 가진다.
- 같은 파일의 user message 생성 경로는 `input.sessionID`를 message info와 part에 넣는다.
- `prompt(...)`는 user message를 생성한 뒤 `loop({ sessionID: input.sessionID })`로 실행 루프에 들어간다.
- `runLoop(sessionID)`는 해당 session의 message history를 읽고 assistant message와 part를 같은 session ID 아래 생성한다.

V2 core도 같은 방향이다.

- `packages/core/src/session/input.ts`: `PromptLifecycle.Admitted`와 `PromptLifecycle.Promoted`가 `sessionID`를 기준으로 durable input과 visible message 승격을 관리한다.
- `packages/core/src/session/run-coordinator.ts`: session별 drain을 조정한다. 같은 session은 같은 drain에 합류하거나 wakeup이 coalescing되고, 다른 session은 병렬 실행될 수 있다.
- `packages/core/src/session/execution/local.ts`: drain 시작 시 `SessionStore.get(sessionID)`로 session location을 찾고 해당 location runtime으로 실행을 배치한다.

이 구조 때문에 session ID는 단순 UI label이 아니라 prompt admission, execution scheduling, event replay, history projection의 기준 키다.

## `/new`와 `/clear` 동작

TUI의 `/new`는 기존 session을 지우지 않는다.

- `packages/tui/src/app.tsx:568` 부근: command name은 `session.new`.
- 같은 command는 `slashName: "new"`와 aliases `["clear"]`를 가진다.
- 실행 내용은 `route.navigate({ type: "home" })`와 `dialog.clear()`다.
- 여기에는 `session.remove`, `session.create`, message delete 같은 API 호출이 없다.

TUI에서 새 ID가 실제로 생기는 지점은 다음 prompt 제출 시점이다.

- `packages/tui/src/component/prompt/index.tsx:986` 부근: `sessionID == null`이면 새 session이 필요하다고 판단한다.
- `packages/tui/src/component/prompt/index.tsx:994` 부근: `sdk.client.session.create(...)`를 호출한다.
- 생성 응답의 `res.data.id`를 이후 prompt 호출의 `sessionID`로 사용한다.
- prompt 전송 뒤 `props.sessionID`가 없던 상태라면 새 session route로 navigate한다.

앱 UI도 같은 패턴이다.

- `packages/app/src/pages/session/use-session-commands.tsx:380` 부근: `session.new` command는 session ID가 없는 `/.../session` 화면으로 navigate한다.
- `packages/app/src/components/prompt-input/submit.ts:326` 부근: `const isNewSession = !params.id`.
- `packages/app/src/components/prompt-input/submit.ts:373` 부근: 새 session이면 `client.session.create()`를 호출한다.
- 이후 생성된 `session.id`로 prompt, shell, command를 실행한다.

따라서 질문의 답은 명확하다. `/new` 또는 `/clear`를 누르는 순간 session ID가 즉시 새로 발급되는 것이 아니라, "현재 session을 떠나 새 session 입력 화면으로 간다"가 정확한 동작이다. 새 ID는 다음 prompt submit에서 만들어진다. 기존 session ID와 이력은 그대로 남는다.

## CLI 실행의 session 생명주기

`opencode run` 계열 CLI는 TUI/app보다 명시적으로 session을 다룬다.

- `packages/opencode/src/cli/cmd/run.ts`: `--session`이 있으면 기존 session을 조회해 그 ID로 실행한다.
- `--fork`가 있으면 기존 session에서 fork API를 호출해 새 session ID를 받는다.
- `--continue`는 최근 session을 찾아 이어간다.
- 아무 옵션도 없으면 `session.create`를 호출해 새 session ID를 만든 뒤 prompt/command를 실행한다.
- one-shot run은 해당 session status가 idle이 되면 프로세스를 종료하지만, session row는 삭제하지 않는다.

즉 "opencode 실행 후 종료까지의 session"이라는 관점에서는, 실행 프로세스의 생명주기와 session row의 생명주기가 분리되어 있다. CLI/TUI/app은 어떤 session ID를 사용할지 선택하고, prompt가 끝나거나 프로세스가 닫히면 client 실행은 끝난다. 그러나 session은 저장소에 남아 추후 이어쓰기 대상이 된다.

## fork, parent, child session

session fork는 새 session ID를 만든 뒤 기존 message/part를 복제하는 별도 경로다.

- `packages/opencode/src/session/session.ts:737` 부근: `fork()`는 `createNext(...)`로 새 session을 만든다.
- 이후 기존 session의 message/part를 새 session ID 아래 복제한다.

`Session.Info` schema와 DB에는 `parentID`/`parent_id`가 존재한다. sub-session, child session, future placement semantics를 담을 수 있는 구조다. 다만 일반 `/new`는 parent-child 관계를 만들지 않고, 기존 session에서 벗어난 새 root session 입력으로 이동하는 UI 명령이다.

## 실무 답변

1. session ID는 어디서 생성되는가?

   `packages/core/src/session/schema.ts`의 `SessionSchema.ID.create()`와 legacy wrapper인 `packages/opencode/src/session/session.ts`의 `SessionID.descending(input.id)` 경로에서 생성된다. 일반 session create API 호출 시 `ses_...` ID가 만들어진다.

2. session ID는 어디에 유지되는가?

   `SessionTable.id` primary key로 DB에 저장되고, message/part/input/context epoch 같은 하위 테이블이 `session_id`로 참조한다. event projector가 생성/수정/삭제 이벤트를 DB projection으로 반영한다.

3. 클라이언트 API에서는 어떤 형태로 사용하는가?

   `POST /session` 응답의 `id`를 받은 뒤 대부분의 operation에서 `/session/{sessionID}/...` URL path parameter로 사용한다. 인증용 header가 아니다.

4. provider/model 요청에서는 어떤 형태로 사용하는가?

   legacy runtime은 outbound model request에 `x-opencode-session`, `x-session-affinity`, `X-Session-Id`, optional `x-parent-session-id`를 넣을 수 있다. provider transform은 session ID를 prompt cache key로 쓸 수 있다.

5. `/new` 또는 `/clear`를 하면 session ID가 새로 생기는가?

   명령 실행 순간에는 아니다. 기존 session을 삭제하거나 비우지도 않는다. UI가 새 session 입력 상태로 이동하고, 다음 prompt submit에서 `session.create`가 호출되면서 새 `ses_...` ID가 생긴다.

6. opencode 종료 시 session은 사라지는가?

   아니다. 프로세스 종료와 session 저장은 분리되어 있다. 명시적 delete/archive/update가 없으면 session row와 message history는 유지된다.
