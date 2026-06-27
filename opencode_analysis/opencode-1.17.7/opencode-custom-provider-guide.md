# opencode Custom Provider 가이드

이 문서는 `opencode`에서 custom provider를 다룰 때 지켜야 하는 규칙을 **3단계 구조**로 정리한 가이드다.

- 1단계: 사용자가 `/connect`로 추가하는 **설정형 custom provider**
- 2단계: 프로젝트의 `opencode.json`에 넣는 **실제 provider 설정 스펙**
- 3단계: 필요할 때 참고하는 **코드 레벨 provider 어댑터 구현 기준**

근거 파일:

- [packages/web/src/content/docs/providers.mdx](D:/AREA51/workspace/opencode/packages/web/src/content/docs/providers.mdx#L2250)
- [packages/app/src/components/dialog-custom-provider-form.ts](D:/AREA51/workspace/opencode/packages/app/src/components/dialog-custom-provider-form.ts#L1)
- [packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx#L1)
- [packages/core/src/config/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts#L1)
- [packages/core/src/config/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/provider.ts#L1)
- [packages/core/src/v1/config/provider-options.ts](D:/AREA51/workspace/opencode/packages/core/src/v1/config/provider-options.ts#L1)
- [packages/llm/src/providers/openai-compatible.ts](D:/AREA51/workspace/opencode/packages/llm/src/providers/openai-compatible.ts#L1)

## 1. 설정형 Custom Provider

이 단계는 `/connect` 명령에서 `Other`를 선택해 provider credential을 저장하는 흐름이다.

### 목적

- `/connect` 목록에 없는 provider를 연결한다.
- API key를 저장한다.
- 이후 `opencode.json`에서 같은 provider ID로 실제 설정을 연결한다.

### UI 흐름

1. `/connect` 실행
2. `Other` 선택
3. provider ID 입력
4. API key 입력
5. `opencode.json`에 provider 설정 추가
6. `/models`에서 모델 선택

이 흐름은 TUI의 provider 선택 다이얼로그와 웹 문서 설명이 일치한다.

### provider ID 규칙

- 소문자/숫자로 시작해야 한다.
- 허용 문자: 소문자, 숫자, `-`, `_`
- `@ai-sdk/` 접두사는 입력 시 제거된다.

예:

- `custom-provider` → 허용
- `custom_provider` → 허용
- `@ai-sdk/custom-provider` → `custom-provider`로 정규화
- `-custom-provider` → 거부
- `Custom Provider` → 거부

### credential 저장 규칙

- `/connect`에서 입력한 API key는 credential로 저장된다.
- 이 credential만으로는 충분하지 않다.
- 실제 동작을 위해서는 프로젝트 설정 파일에 provider를 등록해야 한다.

### 이 단계에서 지켜야 할 점

- provider ID를 나중에 config에서 그대로 사용할 수 있게 정한다.
- API key는 provider credential로 저장되지만, 실제 provider 정의는 따로 필요하다.
- OpenCode가 제공하는 기본 목록이 아닌 provider만 이 경로로 추가하는 것이 맞다.

## 2. `opencode.json` 설정 스펙

이 단계는 custom provider를 실제로 사용할 수 있게 만드는 핵심이다.

### 기본 형태

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "my-model-name": {
          "name": "My Model Display Name"
        }
      }
    }
  }
}
```

### 필수에 가까운 항목

- `provider.<id>.npm`
- `provider.<id>.name`
- `provider.<id>.options.baseURL`
- `provider.<id>.models`

### 권장 값

- OpenAI-compatible API면 `npm: "@ai-sdk/openai-compatible"`
- `/v1/responses` 계열이면 `npm: "@ai-sdk/openai"` 사용 가능

### 선택 항목

- `provider.<id>.options.apiKey`
- `provider.<id>.options.headers`
- `provider.<id>.env`
- `provider.<id>.models.<model-id>.limit`

### 예시: 환경 변수 API key

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1",
        "apiKey": "{env:MY_PROVIDER_API_KEY}",
        "headers": {
          "Authorization": "Bearer custom-token"
        }
      },
      "models": {
        "my-model-name": {
          "name": "My Model Display Name",
          "limit": {
            "context": 200000,
            "output": 65536
          }
        }
      }
    }
  }
}
```

### 예시: 모델이 여러 개일 때

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "chat": {
          "name": "Chat Model"
        },
        "reasoning": {
          "name": "Reasoning Model"
        }
      }
    }
  }
}
```

### 검증 규칙

`DialogCustomProvider`와 `validateCustomProvider()` 기준으로 custom provider 입력값은 다음을 만족해야 한다.

- provider ID는 비어 있으면 안 된다.
- provider ID는 정규식에 맞아야 한다.
- provider name은 필수다.
- baseURL은 필수다.
- baseURL은 `http://` 또는 `https://`로 시작해야 한다.
- model ID는 중복되면 안 된다.
- model name은 필수다.
- header key는 중복되면 안 된다.
- header key/value는 같이 있어야 유효하다.

### 런타임 반영

`ConfigProviderPlugin`이 `opencode.json`의 provider 항목을 읽어서 catalog에 병합한다.

반영 방식은 다음과 같다.

- provider name을 덮어쓴다.
- provider env를 덮어쓴다.
- provider는 `enabled = { via: "custom", data: {} }` 상태가 된다.
- provider api를 덮어쓴다.
- provider request headers/body를 병합한다.
- model family, name, api, capabilities, request, variants, cost, disabled, limit를 병합한다.

즉, custom provider는 단순한 credential 저장이 아니라, **config-driven catalog entry**를 추가하는 방식이다.

## 3. 코드 레벨 Provider 어댑터 기준

이 단계는 “새 provider를 코드로 구현하는 경우”를 위한 참고 기준이다.  
여기는 공식 단일 RFC가 보이진 않아서, 소스상으로 확인되는 범위까지만 정리한다.

### 확인된 핵심 포인트

`packages/core/src/provider.ts` 기준으로 provider는 다음 필드를 가진다.

- `id`
- `name`
- `enabled`
- `env`
- `api`
- `request`

`api`는 두 가지 형태다.

- `aisdk`
- `native`

`request`는 provider 요청 시 사용하는 기본 headers/body를 담는다.

### config provider option lowering

`packages/core/src/v1/config/provider-options.ts`는 provider config를 AI SDK 패키지별로 하위 요청 형태로 변환한다.

예를 들면:

- `@ai-sdk/openai` → OpenAI 방식 lowering
- `@ai-sdk/anthropic` → Anthropic 방식 lowering
- `@ai-sdk/google` → Google 방식 lowering
- `@ai-sdk/azure` → Azure 방식 lowering
- `@ai-sdk/amazon-bedrock` → Bedrock 방식 lowering
- `@ai-sdk/openai-compatible` → OpenAI-compatible lowering

즉, provider 구현에서는 **어떤 AI SDK package를 쓸지**와 **어떻게 provider/request 옵션을 변환할지**가 중요하다.

### OpenAI-compatible provider의 의미

`packages/llm/src/providers/openai-compatible.ts`를 보면 OpenAI-compatible provider는 다음 특징을 가진다.

- `baseURL`을 받는다.
- `apiKey`는 bearer auth로 연결된다.
- `provider` 이름을 선택적으로 덮어쓸 수 있다.
- 동일 패턴의 파생 provider를 만들 수 있다.

예를 들어 `cerebras`, `deepinfra`, `groq`, `mistral`, `togetherai`, `xai`, `openrouter` 같은 provider가 이 계열에 묶여 있다.

### 이 단계에서 지켜야 할 점

- provider ID는 기존 well-known provider와 충돌하지 않게 정한다.
- provider request lowering을 지원해야 한다.
- 모델별 `capabilities`, `limit`, `cost`를 제공하면 UX가 좋아진다.
- `/connect`에서 credential만 저장하고, 실제 provider 정의는 config로 분리하는 구조를 유지하는 것이 좋다.

### 확인 필요

다음은 이 저장소만 봐서는 “단일 공식 문서” 형태로 완결되지 않는다.

- 새 provider를 플러그인으로 추가하는 정식 절차
- provider adapter 구현 시 필수 인터페이스 목록
- provider별 인증 방식 전체 규약
- provider별 request/response normalization 공통 규약

이 부분은 `packages/core/src/plugin/provider/*`, `packages/llm/src/providers/*`, `packages/core/src/provider.ts`, `packages/core/src/config/plugin/provider.ts`를 함께 봐야 한다.

## 4. 실무용 체크리스트

custom provider를 만들 때는 아래 순서를 따르면 된다.

1. provider ID를 정한다.
2. `/connect`로 credential을 저장한다.
3. `opencode.json`에 provider 블록을 추가한다.
4. `npm`을 provider 성격에 맞게 고른다.
5. `baseURL`을 정확히 넣는다.
6. model ID와 model name을 정의한다.
7. 필요하면 `apiKey`, `headers`, `limit`을 추가한다.
8. `/models`에서 노출되는지 확인한다.
9. 실제 요청이 실패하면 `request` lowering과 provider auth를 점검한다.

## 5. 한 줄 정리

`opencode`의 custom provider는 **“credential은 `/connect`에서 저장하고, 실제 provider는 `opencode.json`으로 정의하는 OpenAI-compatible 연결 방식”**이다.

설정형 custom provider의 스펙은 충분히 파악 가능하다.  
반면 코드로 새 provider 어댑터를 만드는 정식 스펙은 저장소 전반에 분산되어 있어, **확인 가능한 범위까지만 추적 가능**하다.
