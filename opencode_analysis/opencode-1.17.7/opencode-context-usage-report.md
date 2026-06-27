# opencode context usage calculation report

## Summary

opencode does not calculate context usage by tokenizing request messages locally. In the default AI SDK runtime, it trusts the usage object emitted by `streamText().fullStream`, normalizes that usage at `finish-step`, stores a non-overlapping token breakdown on the assistant message, and the UI calculates context usage as:

```txt
round((input + output + reasoning + cache.read + cache.write) / model.limit.context * 100)
```

This means a custom adapter that reports prompt character count as `inputTokens` will make opencode display that character count as tokens. If the resulting total is below 0.5% of the model context limit, or if the model context limit cannot be resolved, the UI can still show `0%` while the token number changes.

## Data Flow

1. `packages/opencode/src/session/llm.ts` calls AI SDK `streamText(...)` in the default runtime and converts `fullStream` parts through `LLMAISDK.toLLMEvents`.
2. `packages/opencode/src/session/llm/ai-sdk.ts` extracts usage only from known AI SDK-style fields:
   - `inputTokens`
   - `outputTokens`
   - `totalTokens`
   - `outputTokenDetails.reasoningTokens` or `reasoningTokens`
   - `inputTokenDetails.cacheReadTokens` or `cachedInputTokens`
   - `inputTokenDetails.cacheWriteTokens`
3. `packages/opencode/src/session/processor.ts` applies usage only when it receives a `step-finish` event. It calls `Session.getUsage(...)`, writes the returned `tokens` and `cost` to the assistant message and the `step-finish` part, then persists the message.
4. `packages/opencode/src/session/session.ts` stores session-level token totals in DB columns:
   - `tokens_input`
   - `tokens_output`
   - `tokens_reasoning`
   - `tokens_cache_read`
   - `tokens_cache_write`
5. The app/TUI context indicators do not use API `totalTokens` directly. They recompute a display total from the stored token fields.
6. The context limit used for the percentage is resolved from the public provider/model registry, using the assistant message's `providerID` and `modelID` as lookup keys.

## Server-Side Normalization

`Session.getUsage(...)` in `packages/opencode/src/session/session.ts` normalizes provider usage like this:

```txt
inputTokens = max(0, usage.inputTokens ?? 0)
outputTokens = max(0, usage.outputTokens ?? 0)
reasoningTokens = max(0, usage.reasoningTokens ?? 0)
cacheReadInputTokens = max(0, usage.cacheReadInputTokens ?? 0)
cacheWriteInputTokens = max(0, usage.cacheWriteInputTokens ?? provider metadata fallback ?? 0)

adjustedInputTokens = max(0, inputTokens - cacheReadInputTokens - cacheWriteInputTokens)

tokens.input = adjustedInputTokens
tokens.output = max(0, outputTokens - reasoningTokens)
tokens.reasoning = reasoningTokens
tokens.cache.read = cacheReadInputTokens
tokens.cache.write = cacheWriteInputTokens
```

`usage.totalTokens` is copied to a `total` property on the runtime `tokens` object, but the persisted schemas and UI-facing token shapes only contain `input`, `output`, `reasoning`, and `cache`. In practice, context display is driven by the recomputed field sum, not by `totalTokens`.

Cost is computed from the same normalized categories:

```txt
cost =
  tokens.input * cost.input / 1_000_000
  + tokens.output * cost.output / 1_000_000
  + tokens.cache.read * cost.cache.read / 1_000_000
  + tokens.cache.write * cost.cache.write / 1_000_000
  + tokens.reasoning * cost.output / 1_000_000
```

For cost-tier selection only, opencode uses the raw inclusive `inputTokens` as `contextTokens`.

## UI-Side Context Usage

The app context metric is calculated in `packages/app/src/components/session/session-context-metrics.ts`:

```txt
total = message.tokens.input
  + message.tokens.output
  + message.tokens.reasoning
  + message.tokens.cache.read
  + message.tokens.cache.write

usage = model.limit.context ? round(total / model.limit.context * 100) : null
```

The app chooses the latest assistant message whose token total is greater than zero. The visible indicator then renders `context()?.usage ?? 0`, so a missing usage value can still appear as `0%` in the small indicator/tooltip.

The TUI footer and sidebar use the same total/limit/round formula, but several TUI paths choose the latest assistant message with `tokens.output > 0`. If an adapter only reports input usage and leaves output at zero, the app context tab may show tokens while some TUI usage displays may not update.

## Model Metadata Resolution

The context percentage depends on model metadata, specifically `model.limit.context`. opencode resolves that metadata from configured providers, not from the API response usage object.

### Config File Discovery

When project config is enabled, opencode walks upward from the running directory to the worktree and loads `opencode.jsonc` / `opencode.json`.

Relevant code in `packages/opencode/src/config/paths.ts`:

```ts
export const files = Effect.fn("ConfigPaths.projectFiles")(function* (
  name: string,
  directory: string,
  worktree?: string,
) {
  const afs = yield* FSUtil.Service
  return (yield* afs.up({
    targets: [`${name}.jsonc`, `${name}.json`],
    start: directory,
    stop: worktree,
  })).toReversed()
})
```

Relevant call site in `packages/opencode/src/config/config.ts`:

```ts
if (!Flag.OPENCODE_DISABLE_PROJECT_CONFIG) {
  for (const file of yield* ConfigPaths.files("opencode", ctx.directory, ctx.worktree).pipe(Effect.orDie)) {
    yield* merge(file, yield* loadFile(file, authEnv), "local")
  }
}
```

So placing `opencode.json` in the directory where opencode is launched is valid, as long as project config is not disabled.

### Config Schema

Top-level model selection is a string in `provider/model` format:

```ts
model: Schema.optional(Schema.String).annotate({
  description: "Model to use in the format of provider/model, eg anthropic/claude-2",
})
```

Provider definitions live under the top-level `provider` object:

```ts
provider: Schema.optional(Schema.Record(Schema.String, ConfigProviderV1.Info)).annotate({
  description: "Custom provider configurations and model overrides",
})
```

Each configured model may define its own `limit`:

```ts
limit: Schema.optional(
  Schema.Struct({
    context: Schema.Finite,
    input: Schema.optional(Schema.Finite),
    output: Schema.Finite,
  }),
)
```

### Provider Registry Construction

When opencode builds the provider registry, each configured model is stored under the `models` map key from `opencode.json`. That map key becomes the internal `modelID` used by session messages and UI lookup.

Relevant code in `packages/opencode/src/provider/provider.ts`:

```ts
for (const [modelID, model] of Object.entries(provider.models ?? {})) {
  const existingModel = parsed.models[model.id ?? modelID]
  const apiID = model.id ?? existingModel?.api.id ?? modelID
  const apiNpm =
    model.provider?.npm ??
    provider.npm ??
    existingModel?.api.npm ??
    modelsDev[providerID]?.npm ??
    "@ai-sdk/openai-compatible"

  const parsedModel: Model = {
    id: ModelV2.ID.make(modelID),
    api: {
      id: apiID,
      npm: apiNpm,
      url: model.provider?.api ?? provider?.api ?? existingModel?.api.url ?? modelsDev[providerID]?.api ?? "",
    },
    providerID: ProviderV2.ID.make(providerID),
    limit: {
      context: model.limit?.context ?? existingModel?.limit?.context ?? 0,
      input: model.limit?.input ?? existingModel?.limit?.input,
      output: model.limit?.output ?? existingModel?.limit?.output ?? 0,
    },
  }
  parsed.models[modelID] = parsedModel
}
```

Important distinction:

- `modelID` map key: opencode's internal model ID, used for `provider.models[modelID]` lookup.
- `model.id` inside config: optional upstream/API model ID, stored as `model.api.id`, used when calling the provider SDK.

This means the following config selects internal model `workflow`, while sending `actual-api-model-name` to the provider:

```json
{
  "model": "aiu/workflow",
  "provider": {
    "aiu": {
      "name": "AIU",
      "npm": "@ai-sdk/openai-compatible",
      "api": "http://localhost:3000/v1",
      "models": {
        "workflow": {
          "id": "actual-api-model-name",
          "name": "AIU Workflow",
          "limit": {
            "context": 400000,
            "output": 8192
          }
        }
      }
    }
  }
}
```

### Model Selection

The selected model string is parsed by splitting on the first slash:

```ts
export function parseModel(model: string) {
  const [providerID, ...rest] = model.split("/")
  return {
    providerID: ProviderV2.ID.make(providerID),
    modelID: ModelV2.ID.make(rest.join("/")),
  }
}
```

The default model resolver returns `parseModel(cfg.model)` when `model` is present in config:

```ts
const defaultModel = Effect.fn("Provider.defaultModel")(function* () {
  const cfg = yield* config.get()
  if (cfg.model) return parseModel(cfg.model)
  ...
})
```

Session prompt creation stores the selected `providerID/modelID` on the user message:

```ts
const info: SessionV1.User = {
  ...
  model: {
    providerID: model.providerID,
    modelID: model.modelID,
    variant,
  },
  ...
}
```

Before running the provider request, opencode resolves that internal ID through the provider registry:

```ts
const getModel = Effect.fn("SessionPrompt.getModel")(function* (
  providerID: ProviderV2.ID,
  modelID: ModelV2.ID,
  sessionID: SessionID,
) {
  const exit = yield* provider.getModel(providerID, modelID).pipe(Effect.exit)
  if (Exit.isSuccess(exit)) return exit.value
  ...
})
```

The provider service performs an exact map lookup:

```ts
const getModel = Effect.fn("Provider.getModel")(function* (providerID: ProviderV2.ID, modelID: ModelV2.ID) {
  const s = yield* InstanceState.get(state)
  const provider = s.providers[providerID]
  if (!provider) {
    return yield* new ModelNotFoundError({ providerID, modelID, suggestions })
  }

  const info = provider.models[modelID]
  if (!info) {
    return yield* new ModelNotFoundError({ providerID, modelID, suggestions })
  }
  return info
})
```

The provider SDK call uses `model.api.id`, not the internal map key:

```ts
const language = s.modelLoaders[model.providerID]
  ? await s.modelLoaders[model.providerID](sdk, model.api.id, { ...provider.options, ...model.options }, model)
  : sdk.languageModel(model.api.id)
```

### Assistant Message IDs Used by the UI

The assistant message stores `modelID` from the resolved model object. For configured models, that is the internal `models` map key:

```ts
const assistantMessage: SessionV1.Assistant = yield* sessions.updateMessage({
  ...
  modelID: taskModel.id,
  providerID: taskModel.providerID,
  ...
})
```

The app's context usage metric then uses the assistant message fields to look up metadata:

```ts
const provider = providers.find((item) => item.id === message.providerID)
const model = provider?.models[message.modelID]
const limit = model?.limit.context
const total = tokenTotal(message)

return {
  ...
  usage: limit ? Math.round((total / limit) * 100) : null,
}
```

Therefore, for usage percentage to be meaningful:

```txt
assistantMessage.providerID === provider id in provider registry
assistantMessage.modelID === key under provider.models
provider.models[assistantMessage.modelID].limit.context > 0
```

### Common Configuration Trap

This can look correct at first glance but is wrong for opencode's metadata lookup:

```json
{
  "model": "aiu/actual-api-model-name",
  "provider": {
    "aiu": {
      "models": {
        "workflow": {
          "id": "actual-api-model-name",
          "limit": {
            "context": 400000,
            "output": 8192
          }
        }
      }
    }
  }
}
```

Here the selected internal model ID is `actual-api-model-name`, but the configured model metadata is stored under `provider.models.workflow`. The context usage UI will not find `provider.models["actual-api-model-name"]`.

Use this instead:

```json
{
  "model": "aiu/workflow",
  "provider": {
    "aiu": {
      "models": {
        "workflow": {
          "id": "actual-api-model-name",
          "limit": {
            "context": 400000,
            "output": 8192
          }
        }
      }
    }
  }
}
```

In this valid shape:

```txt
opencode internal modelID = workflow
provider API model id     = actual-api-model-name
context limit lookup      = provider.models.workflow.limit.context
```

## Why Tokens Can Update While Usage Stays 0%

The observed behavior is consistent with these cases:

1. **Integer rounding below 0.5%.** The UI uses `Math.round`. For a model with a 400,000 context limit, any total below 2,000 displays as `0%`; for 1,000,000, anything below 5,000 displays as `0%`.
2. **Missing or mismatched model metadata.** If the assistant message `providerID/modelID` does not resolve to `providers[].models[modelID].limit.context`, the app indicator falls back to `0%` even though token totals are present. This is especially likely when `opencode.json` uses the upstream API model ID in top-level `"model"` but stores metadata under a different `provider.models` map key.
3. **Unit mismatch.** If aiu-opencode-adapter reports character count as `inputTokens`, opencode treats characters as tokens. The displayed token count updates, but the percentage is characters divided by a token context limit. This is not semantically meaningful unless the model limit is also configured in the same unit.
4. **Output tokens omitted.** Some TUI displays require `tokens.output > 0` to select the latest assistant message, so input-only usage can produce inconsistent UI surfaces.
5. **`totalTokens` alone is insufficient.** opencode's UI does not use stored `totalTokens`; it uses the per-category token fields. An adapter should populate `inputTokens` and `outputTokens` rather than relying on `totalTokens`.

## Implications for aiu-opencode-adapter

If the adapter currently returns the workflow input message character count as `inputTokens`, opencode will:

- store that value as `tokens.input` after cache subtraction;
- include it in the UI total;
- use it for compaction/overflow checks against model context limits;
- use it for cost estimation as if it were token count;
- display `0%` whenever the rounded percentage is below 1%, or when the model limit is unavailable.

For a stable integration, the adapter should return usage in opencode/AI SDK semantics:

```txt
inputTokens: prompt token estimate, inclusive of cache if any
outputTokens: completion token estimate
totalTokens: inputTokens + outputTokens, optional but useful for audit
inputTokenDetails.cacheReadTokens/cacheWriteTokens: only when known
outputTokenDetails.reasoningTokens or reasoningTokens: only when known
```

If exact tokenization is not available, a conservative token estimate is better than raw character count for opencode's context percentage, compaction, and cost logic. If the workflow hard limit is truly character-based and the adapter intentionally reports characters, the corresponding model `limit.context` should be understood as a character-limit proxy too, but that will make costs and token labels misleading.

## Files Reviewed

- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/session/llm/ai-sdk.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/session.ts`
- `packages/opencode/src/session/overflow.ts`
- `packages/opencode/src/config/config.ts`
- `packages/opencode/src/config/paths.ts`
- `packages/opencode/src/provider/provider.ts`
- `packages/opencode/src/session/prompt.ts`
- `packages/core/src/v1/config/config.ts`
- `packages/core/src/v1/config/provider.ts`
- `packages/core/src/session/sql.ts`
- `packages/core/src/session/projector.ts`
- `packages/core/src/session/message.ts`
- `packages/app/src/components/session/session-context-metrics.ts`
- `packages/app/src/components/session-context-usage.tsx`
- `packages/tui/src/component/prompt/index.tsx`
- `packages/tui/src/feature-plugins/sidebar/context.tsx`
