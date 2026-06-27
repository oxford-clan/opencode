# opencode Custom Provider Spec

이 문서는 `opencode`의 **custom provider** 설정과 연결 흐름만 따로 정리한 스펙 문서입니다.

근거 소스:

- [packages/web/src/content/docs/providers.mdx](D:/AREA51/workspace/opencode/packages/web/src/content/docs/providers.mdx#L2250)
- [packages/app/src/components/dialog-custom-provider-form.ts](D:/AREA51/workspace/opencode/packages/app/src/components/dialog-custom-provider-form.ts#L1)
- [packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx#L1)
- [packages/core/src/config/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts#L1)
- [packages/core/src/config/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/provider.ts#L1)
- [packages/core/test/config/provider.test.ts](D:/AREA51/workspace/opencode/packages/core/test/config/provider.test.ts#L1)
- [packages/opencode/test/cli/cmd/tui/provider-options.test.ts](D:/AREA51/workspace/opencode/packages/opencode/test/cli/cmd/tui/provider-options.test.ts#L1)

## 1. 개요

`custom provider`는 `/connect`에서 목록에 없는 **OpenAI-compatible provider**를 직접 추가하는 기능이다.

핵심 목적은 다음과 같다.

- OpenCode에 기본 포함되지 않은 OpenAI-compatible API를 연결한다.
- provider별 API key를 저장한다.
- 프로젝트의 `opencode.json`에 provider 설정을 추가해 런타임에서 실제 모델 목록과 요청 옵션을 정의한다.

즉, `/connect` 단계는 **credential 저장**, `opencode.json` 단계는 **provider/model 구성**이다.

## 2. 사용자 흐름

### TUI에서의 흐름

`/connect` 명령을 열면 provider 목록 맨 아래에 `Other` 항목이 있다.

1. `/connect` 실행
2. `Other` 선택
3. provider ID 입력
4. API key 입력
5. `opencode.json`에 provider 설정 추가
6. `/models` 실행해서 새 provider와 모델을 선택

이 흐름은 [packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx#L1)와 [packages/web/src/content/docs/providers.mdx](D:/AREA51/workspace/opencode/packages/web/src/content/docs/providers.mdx#L2250)에서 일치한다.

### 중요한 UX 의미

- `/connect`에서 입력한 API key만으로는 충분하지 않다.
- custom provider는 **반드시** `opencode.json`에 같은 provider ID로 설정해야 한다.
- UI는 credential 저장과 config 작성 단계를 분리한다.

## 3. provider ID 규칙

provider ID는 다음 규칙을 따른다.

- 소문자/숫자로 시작해야 한다.
- 허용 문자: 소문자, 숫자, `-`, `_`
- `@ai-sdk/` 접두사는 입력 시 제거된다.

구현 근거:

- `normalizeCustomProviderID()`는 `@ai-sdk/`를 제거한 뒤 정규식 검증을 수행한다.
- 정규식은 `/^[a-z0-9][a-z0-9-_]*$/`이다.

관련 파일:

- [packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx#L1)
- [packages/opencode/test/cli/cmd/tui/provider-options.test.ts](D:/AREA51/workspace/opencode/packages/opencode/test/cli/cmd/tui/provider-options.test.ts#L1)

## 4. `/connect`에서 저장되는 내용

custom provider를 연결할 때 저장되는 것은 두 가지다.

### 4.1 credential 저장

`ApiMethod`에서 입력한 API key는 다음과 같이 저장된다.

- `sdk.client.auth.set(...)`
- `auth.type = "api"`
- `auth.key = <API key>`

`DialogCustomProvider`는 입력된 API key를 저장한 뒤, 같은 provider ID에 대한 설정을 다시 불러온다.

### 4.2 config 저장

custom provider의 설정은 `serverSync.updateConfig(...)`를 통해 `provider` 섹션에 반영된다.

관련 파일:

- [packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/component/dialog-provider.tsx#L1)
- [packages/app/src/components/dialog-custom-provider.tsx](D:/AREA51/workspace/opencode/packages/app/src/components/dialog-custom-provider.tsx#L1)

## 5. UI 레벨의 설정 폼

`DialogCustomProvider` 폼에서 입력받는 필드는 다음과 같다.

- `providerID`
- `name`
- `baseURL`
- `apiKey`
- `models[]`
- `headers[]`

각 모델 항목은 `id`, `name`을 가진다.  
각 헤더 항목은 `key`, `value`를 가진다.

폼 검증은 [packages/app/src/components/dialog-custom-provider-form.ts](D:/AREA51/workspace/opencode/packages/app/src/components/dialog-custom-provider-form.ts#L1)에서 수행된다.

## 6. 검증 규칙

### provider ID

- 필수
- 정규식 불일치 시 오류
- 이미 존재하는 provider ID면 오류
- 단, `disabled_providers`에 포함된 경우는 재사용 가능

### name

- 필수

### baseURL

- 필수
- `http://` 또는 `https://`로 시작해야 한다

### models

- 각 model은 `id`, `name`이 필요하다
- 빈 값은 허용되지 않는다
- 같은 provider 안에서 model ID는 중복될 수 없다

### headers

- key/value 쌍이 모두 있어야 유효하다
- key 중복은 허용되지 않는다
- 대소문자 구분 없이 중복 검사한다
- key와 value가 모두 비어 있으면 해당 row는 무시된다

### apiKey

- 빈 값 가능
- `{env:VAR_NAME}` 형식이면 environment variable 참조로 해석한다
- 그 외 값은 직접 API key로 저장된다

관련 파일:

- [packages/app/src/components/dialog-custom-provider-form.ts](D:/AREA51/workspace/opencode/packages/app/src/components/dialog-custom-provider-form.ts#L1)

## 7. `opencode.json` 설정 스키마

custom provider는 프로젝트의 `opencode.json`에서 다음 형태로 정의한다.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider Display Name",
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

### 허용 옵션

문서와 구현을 합치면 custom provider에서 실질적으로 중요한 옵션은 다음이다.

- `npm`
- `name`
- `env`
- `options.baseURL`
- `options.apiKey`
- `options.headers`
- `models`

문서상 기본 권장값은 다음과 같다.

- OpenAI-compatible provider라면 `npm: "@ai-sdk/openai-compatible"`
- `/v1/responses` 계열이면 `npm: "@ai-sdk/openai"` 사용 가능

관련 파일:

- [packages/web/src/content/docs/providers.mdx](D:/AREA51/workspace/opencode/packages/web/src/content/docs/providers.mdx#L2250)
- [packages/core/src/config/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/provider.ts#L1)

## 8. 고급 예시

API key를 환경 변수로 넣고, 커스텀 헤더와 모델 limit까지 설정할 수 있다.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My AI Provider Display Name",
      "options": {
        "baseURL": "https://api.myprovider.com/v1",
        "apiKey": "{env:ANTHROPIC_API_KEY}",
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

이 예시에서 확인되는 동작:

- `apiKey`는 `{env:...}` 형식이면 환경 변수 참조로 저장된다.
- `headers`는 각 요청에 추가되는 커스텀 헤더다.
- `limit.context`와 `limit.output`은 모델의 최대 컨텍스트/출력 크기를 의미한다.

관련 파일:

- [packages/web/src/content/docs/providers.mdx](D:/AREA51/workspace/opencode/packages/web/src/content/docs/providers.mdx#L2250)
- [packages/core/test/config/provider.test.ts](D:/AREA51/workspace/opencode/packages/core/test/config/provider.test.ts#L1)

## 9. 런타임에서의 해석 방식

`opencode.json`의 provider 설정은 `ConfigProviderPlugin`이 읽어서 catalog에 반영한다.

반영 규칙은 다음과 같다.

- provider의 `name`이 있으면 덮어쓴다.
- provider의 `env`가 있으면 덮어쓴다.
- provider는 `enabled = { via: "custom", data: {} }` 상태가 된다.
- provider의 `api`가 있으면 덮어쓴다.
- provider의 `request.headers`와 `request.body`는 누적 병합된다.
- model의 `family`, `name`, `api`, `capabilities`, `request`, `variants`, `cost`, `disabled`, `limit`도 개별적으로 병합된다.

관련 파일:

- [packages/core/src/config/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts#L1)

## 10. 구현상 주의점

### 10.1 custom provider는 OpenAI-compatible 중심이다

문서상 custom provider는 기본적으로 **OpenAI-compatible provider**를 위한 진입점이다.

- `/connect`의 `Other`는 일반 provider 목록 밖의 provider를 추가하기 위한 선택지다.
- config 예시의 기본값은 `@ai-sdk/openai-compatible`이다.

### 10.2 credential과 config는 별개다

UI에서 API key를 저장해도, `opencode.json`에 provider 설정이 없으면 실제로 사용되지 않을 수 있다.

문서도 이 점을 명시한다:

- “This only stores a credential ... you will need to configure the provider in opencode.json”

### 10.3 모델 ID는 provider 내부에서만 유일하면 된다

모델 저장 구조는 provider 하위에 중첩된다.  
즉, 모델 ID는 전역이 아니라 **provider 내부에서만** 유일성을 가진다.

관련 파일:

- [packages/core/src/config/plugin/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts#L1)
- [packages/core/src/provider.ts](D:/AREA51/workspace/opencode/packages/core/src/provider.ts#L1)

## 11. 요약

custom provider 스펙을 한 문장으로 정리하면 다음과 같다.

> `/connect`에서 provider ID와 API key를 저장하고, 프로젝트의 `opencode.json`에서 `@ai-sdk/openai-compatible` 기반 provider와 모델을 정의해 OpenCode에 등록하는 방식이다.

핵심 포인트는 다음이다.

- provider ID는 소문자/숫자/`-`/`_`만 허용
- `/connect`는 credential 저장용
- `opencode.json`은 실제 provider/model 구성용
- 기본 구현은 OpenAI-compatible provider 기준
- `baseURL`, `headers`, `apiKey`, `models`, `limit`을 커스터마이즈할 수 있음
- runtime에서는 `ConfigProviderPlugin`이 이를 catalog에 병합함
