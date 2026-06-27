# opencode Context 자동 압축 분석 보고서

> 분석 대상: `D:\AREA51\workspace\opencode`  
> 작성 일자: 2026-06-20  
> 관련 문서: `opencode-context-usage-report.md`, `opencode-system-prompt-swap-cases.md`

---

## 1. 결론

opencode의 context 자동 압축은 화면에 보이는 `usage %`가 특정 고정값을 넘었는지로 판단하지 않는다.

핵심 기준은 다음이다.

```txt
현재 대화 토큰 count >= 모델별 사용 가능 토큰 한도
```

따라서 실제 자동 압축 시점은 모델의 `context`, `input`, `output` limit과 `compaction.reserved` 또는 V2의 `buffer` 설정에 따라 달라진다. UI usage는 단순 표시값이고, 자동 압축은 별도의 overflow 계산식을 사용한다.

---

## 2. 기존 context usage 표시와의 차이

기존 `opencode-context-usage-report.md`에서 정리한 UI usage 계산식은 다음이다.

```txt
round((input + output + reasoning + cache.read + cache.write) / model.limit.context * 100)
```

이 값은 app/TUI가 context 사용량을 보여주기 위한 표시값이다. 자동 압축은 이 표시값을 직접 읽지 않는다.

자동 압축은 `packages/opencode/src/session/overflow.ts`의 `isOverflow()`를 통해 별도로 계산된다.

```ts
const count =
  input.tokens.total || input.tokens.input + input.tokens.output + input.tokens.cache.read + input.tokens.cache.write
return count >= usable(input)
```

주의할 점:

- `tokens.total`이 있으면 우선 사용한다.
- 없으면 `input + output + cache.read + cache.write`를 더한다.
- 여기에는 UI usage 계산에 포함되는 `reasoning`이 직접 더해지지 않는다.
- `model.limit.context === 0`이면 자동 압축은 동작하지 않는다.
- `compaction.auto === false`이면 자동 압축은 동작하지 않는다.

---

## 3. v1 자동 압축 임계값

v1 경로의 핵심 코드는 `packages/opencode/src/session/overflow.ts`다.

```ts
const COMPACTION_BUFFER = 20_000

export function usable(input) {
  const context = input.model.limit.context
  if (context === 0) return 0

  const reserved =
    input.cfg.compaction?.reserved ??
    Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))
  return input.model.limit.input
    ? Math.max(0, input.model.limit.input - reserved)
    : Math.max(0, context - ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))
}
```

`ProviderTransform.maxOutputTokens()`의 기본 상수는 다음이다.

```ts
export const OUTPUT_TOKEN_MAX = 32_000

export function maxOutputTokens(model, outputTokenMax = OUTPUT_TOKEN_MAX) {
  return Math.min(model.limit.output, outputTokenMax) || outputTokenMax
}
```

즉 v1 자동 압축 기준은 다음처럼 환산할 수 있다.

### `model.limit.input`이 없는 경우

```txt
usable = context - maxOutputTokens
maxOutputTokens = min(model.limit.output, 32,000)

자동 압축 표시상 기준 % ~= usable / context * 100
```

### `model.limit.input`이 있는 경우

```txt
reserved = compaction.reserved ?? min(20,000, maxOutputTokens)
usable = input - reserved

자동 압축 표시상 기준 % ~= usable / context * 100
```

중요한 구현상 차이:

- `model.limit.input`이 있을 때는 `compaction.reserved`가 사용된다.
- `model.limit.input`이 없을 때는 `reserved`를 계산하지만 실제 반환식은 `context - maxOutputTokens`를 사용한다.

---

## 4. 사용자가 제시한 모델 설정의 자동 압축 기준

설정:

```json
{
  "claude-opus-4-6": {
    "name": "Claude Opus 4.6",
    "limit": { "context": 100000, "output": 16000 }
  }
}
```

이 설정에는 `limit.input`이 없다. 따라서 v1 기준은 `context - maxOutputTokens`다.

```txt
context = 100,000
output = 16,000
OUTPUT_TOKEN_MAX = 32,000

maxOutputTokens = min(16,000, 32,000)
                = 16,000

usable = 100,000 - 16,000
       = 84,000

84,000 / 100,000 * 100 = 84%
```

따라서 일반 v1 provider model 설정 기준으로는 assistant step usage count가 `84,000` 이상이 되는 시점, 즉 표시상 약 `84%`에서 자동 압축이 예약된다.

주의:

- UI는 반올림하므로 `83,500` tokens부터도 화면에는 `84%`로 보일 수 있다.
- 실제 조건은 표시값 `84%`가 아니라 내부 count `>= 84,000`이다.

---

## 5. 자동 압축이 실행되는 시점

자동 압축은 assistant 응답 도중 또는 직후에 예약된다.

주요 경로:

1. `packages/opencode/src/session/processor.ts`
   - provider stream의 `step-finish` 이벤트에서 usage를 정규화하고 assistant message에 저장한다.
   - summary message가 아니고 `isOverflow()`가 true이면 `ctx.needsCompaction = true`를 설정한다.
   - `Stream.takeUntil(() => ctx.needsCompaction)`로 현재 stream 처리를 끊고 `"compact"` 결과를 반환한다.

2. `packages/opencode/src/session/prompt.ts`
   - processor가 `"compact"`를 반환하면 `compaction.create({ auto: true })`를 호출한다.
   - 다음 loop에서 `task.type === "compaction"`을 처리하면서 실제 요약을 수행한다.

3. provider가 context overflow 에러를 던진 경우
   - `ContextOverflowError`가 발생해도 `compaction.auto !== false`이면 `ctx.needsCompaction = true`로 전환된다.
   - 이 경우에도 compaction task가 생성된다.

자동 압축 비활성화 조건:

- config에서 `compaction.auto: false`
- 환경 플래그 `OPENCODE_DISABLE_AUTOCOMPACT`
- model metadata의 `limit.context`가 `0`

---

## 6. `/compact` 명령의 실제 동작

`/compact`는 자동 압축과 같은 compaction 요약 파이프라인을 사용하지만, 서버에는 `auto: false`로 들어간다.

진입 경로:

- TUI: `packages/tui/src/routes/session/index.tsx`
  - `session.compact` command
  - slash command 이름은 `compact`
  - alias는 `summarize`
  - 내부적으로 `sdk.client.session.summarize(...)` 호출

- App: `packages/app/src/pages/session/use-session-commands.tsx`
  - `session.compact`
  - 내부적으로 `sdk().client.session.summarize(...)` 호출

- Server: `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`
  - `summarize` handler
  - 마지막 user message의 agent를 찾아 compaction marker를 생성
  - `auto: ctx.payload.auto ?? false`
  - `promptSvc.loop({ sessionID })` 실행

서버가 만드는 marker는 일반 user message가 아니라 `type: "compaction"` part를 가진 내부 작업 메시지다.

```ts
yield* session.updatePart({
  type: "compaction",
  auto: input.auto,
  overflow: input.overflow,
})
```

이 marker를 prompt loop가 발견하면 `compaction.process()`를 실행한다.

---

## 7. compaction 처리 내용

`packages/opencode/src/session/compaction.ts`가 실제 요약을 수행한다.

핵심 동작:

- hidden `compaction` agent를 사용한다.
- 해당 agent의 prompt는 `packages/opencode/src/agent/prompt/compaction.txt`다.
- tools는 빈 객체로 전달된다.
- 오래된 message head를 요약 대상으로 삼는다.
- 최근 tail은 가능한 한 원문 그대로 보존한다.
- 기존 compaction summary가 있으면 새 history와 병합해서 갱신한다.
- assistant summary message를 만들고 `summary: true`, `mode: "compaction"`, `agent: "compaction"`으로 저장한다.

기본 최근 tail 보존 기준:

```txt
tail_turns 기본값 = 2
preserve_recent_tokens 기본값 =
  min(8,000, max(2,000, floor(usable * 0.25)))
```

tool output은 compaction 요약 입력에서 최대 2,000자까지 줄인다.

```txt
TOOL_OUTPUT_MAX_CHARS = 2,000
```

요약 성공 후 model context에 전달되는 메시지는 `filterCompacted()`를 통해 재배치된다.

```txt
[compaction-user, summary, ...retained tail, ...newer messages]
```

즉 오래된 전체 대화가 그대로 남는 것이 아니라, summary message와 최근 tail 중심으로 다음 provider request가 구성된다.

---

## 8. 자동 압축과 수동 `/compact`의 차이

자동 압축:

- `auto: true`
- overflow 직후 작업을 이어가기 위해 synthetic user message를 만들 수 있다.
- synthetic text:

```txt
Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed.
```

- overflow가 attachment 크기 문제였으면 media 제거 안내 문구가 앞에 붙는다.
- 원래 overflow를 일으킨 user message를 replay할 수 있는 경우 replay message를 생성한다.

수동 `/compact`:

- `auto: false`
- 사용자가 직접 요약을 요청한 것이므로 자동 continue synthetic user message를 만들지 않는다.
- 요약이 끝나면 다음 사용자 입력을 기다리는 흐름이다.

---

## 9. V2 Session Core의 자동 압축 기준

V2 경로는 `packages/core/src/session/compaction.ts`에 별도 구현이 있다.

기본 설정:

```txt
DEFAULT_BUFFER = 20,000
DEFAULT_KEEP_TOKENS = 8,000
SUMMARY_OUTPUT_TOKENS = 4,096
```

설정 스키마는 `packages/core/src/config/compaction.ts`에 있으며 이름이 v1과 다르다.

```ts
{
  auto?: boolean
  prune?: boolean
  keep?: { tokens?: number }
  buffer?: number
}
```

V2 자동 압축 기준:

```txt
estimate(system + messages + tools) > context - max(output, buffer)
```

즉 기본값 기준으로는 다음이다.

```txt
자동 압축 기준 % ~= (context - max(output, 20,000)) / context * 100
```

사용자가 제시한 `context=100,000`, `output=16,000` 모델을 V2 공식에 적용하면:

```txt
context - max(16,000, 20,000)
= 100,000 - 20,000
= 80,000

80,000 / 100,000 * 100 = 80%
```

따라서 V2 runner 기준이면 약 `80%` 초과에서 압축 시도가 일어난다. 다만 현재 사용자가 제시한 provider model limit 설정을 일반 v1 경로로 해석하면 기준은 `84%`다.

---

## 10. prune 옵션

`compaction.prune`은 summary compaction과 별도 기능이다.

`packages/opencode/src/session/compaction.ts`에는 오래된 tool output을 제거하는 `prune()` 로직이 있다.

기본 상수:

```txt
PRUNE_MINIMUM = 20,000
PRUNE_PROTECT = 40,000
```

동작 개요:

- `cfg.compaction.prune`이 true일 때만 실행된다.
- 뒤에서부터 tool output을 훑으면서 최근 약 40,000 token 상당의 tool output은 보호한다.
- 그보다 오래된 tool output을 compacted 처리해 다음 context에서 줄인다.
- 기본값은 false다.

이 기능은 `/compact` 요약과는 다르며, 자동 압축 임계값 자체를 결정하지 않는다.

---

## 11. aiu-opencode-adapter 관점 시사점

1. adapter가 반환하는 usage 값은 자동 압축에도 영향을 준다.
   - v1 `isOverflow()`는 assistant message의 token usage를 기반으로 한다.
   - adapter가 문자 수를 `inputTokens`로 반환하면 opencode는 그것을 token count처럼 사용한다.

2. model limit metadata가 중요하다.
   - `model.limit.context`가 0이거나 lookup이 실패하면 자동 압축과 UI usage 모두 의미 있게 동작하지 않을 수 있다.

3. UI usage와 자동 압축 기준은 다르다.
   - UI가 84%라고 보여도 내부 count가 기준 미만이면 압축이 실행되지 않을 수 있다.
   - 반대로 V2처럼 provider 호출 전 추정 기준을 쓰는 경로에서는 UI의 마지막 assistant usage보다 먼저 압축이 시도될 수 있다.

4. compaction 요청은 일반 대화 요청과 system prompt가 다르다.
   - 기존 `opencode-system-prompt-swap-cases.md`에서 정리한 것처럼 `compaction`은 hidden agent이며 `agent/prompt/compaction.txt`를 base prompt로 쓴다.
   - adapter 로그에서 일반 agent 요청과 compaction 요청의 system prompt가 달라지는 것은 정상이다.

---

## 12. 근거 파일 색인

| 관심사 | 위치 |
| --- | --- |
| v1 자동 압축 임계값 계산 | `packages/opencode/src/session/overflow.ts` |
| 기본 output token max | `packages/opencode/src/provider/transform.ts` |
| runtime output max override | `packages/opencode/src/effect/runtime-flags.ts` |
| assistant step-finish usage 저장 및 overflow 검사 | `packages/opencode/src/session/processor.ts` |
| prompt loop의 compaction task 처리 | `packages/opencode/src/session/prompt.ts` |
| compaction marker 생성 및 요약 처리 | `packages/opencode/src/session/compaction.ts` |
| compacted history 재배치 | `packages/opencode/src/session/message-v2.ts` |
| `/compact` TUI command | `packages/tui/src/routes/session/index.tsx` |
| `/compact` App command | `packages/app/src/pages/session/use-session-commands.tsx` |
| HTTP summarize handler | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` |
| compaction hidden agent 정의 | `packages/opencode/src/agent/agent.ts` |
| compaction prompt | `packages/opencode/src/agent/prompt/compaction.txt` |
| v1 config schema | `packages/core/src/v1/config/config.ts` |
| config docs | `packages/web/src/content/docs/ko/config.mdx` |
| V2 compaction logic | `packages/core/src/session/compaction.ts` |
| V2 compaction config schema | `packages/core/src/config/compaction.ts` |
| V2 LLM runner compaction call site | `packages/core/src/session/runner/llm.ts` |

