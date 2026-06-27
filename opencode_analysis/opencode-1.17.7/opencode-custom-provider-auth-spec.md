# OpenCode 커텀 프로바이더 인증(AUTH KEY) 강제화 검토 보고서

이 보고서는 `opencode.json`에 커스텀 프로바이더(Custom Provider)를 등록하여 모델을 전환할 때, 별도의 인증 키(API Key) 입력을 요구하지 않고 인증 헤더 없이 요청이 발송되어 401(Unauthorized) 에러를 일으키는 현상을 분석하고, 커스텀 프로바이더 사용 시에도 API 인증 키 입력을 강제하도록 구현하는 방법 및 소스코드 수정안을 제시합니다.

---

## 1. 현상 및 원인 분석 (Root Cause Analysis)

### 1) 현상
* `opencode.json`에 커스텀 프로바이더를 추가한 뒤 해당 모델을 선택하면, 기본 탑재 프로바이더(Anthropic, OpenAI 등)와 달리 **인증 키 등록(AUTH KEY) 다이얼로그나 프롬프트가 뜨지 않고 즉시 선택**됩니다.
* 이 상태에서 프롬프트를 전송하면 API Key 누락으로 인해 연결 실패 오류가 발생합니다.

### 2) 소스코드 레벨 원인 분석
* **핵심 코드**: `available` 여부 판정 로직 ([packages/core/src/catalog.ts](file:///D:/AREA51/workspace/opencode/packages/core/src/catalog.ts#L96-L101))
  ```ts
  const available = (provider: ProviderV2.Info, integration: Integration.Info | undefined, connected: boolean) => {
    if (provider.disabled) return false
    if (typeof provider.request.body.apiKey === "string") return true
    if (connected) return true
    return !integration // 💡 문제의 원인 지점
  }
  ```
  * `integration` 객체가 존재하지 않으면 (`!integration`이 `true`), 해당 프로바이더가 별도의 인증을 필요로 하지 않는 공개 API 등으로 판단하여 **연결 여부(connected)에 관계없이 무조건 사용 가능(available = true)하다고 판정**합니다.

* **동기화 플러그인 로직**: 프로바이더 정보 조립 플러그인 ([packages/core/src/config/plugin/provider.ts](file:///D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts#L27-L43))
  ```ts
  yield* integrationTransform((integrations) => {
    for (const file of files) {
      for (const [id, item] of Object.entries(file.info.providers ?? {})) {
        const integrationID = Integration.ID.make(id)
        if (!configuredIntegrations.has(id) && !integrations.get(integrationID)) continue
        integrations.update(integrationID, (integration) => {
          integration.name = item.name ?? integration.name
        })
        if (item.env !== undefined) {
          integrations.method.update({
            integrationID,
            method: { type: "env", names: [...item.env] },
          })
        }
      }
    }
  })
  ```
  * `opencode.json`에 선언된 커스텀 프로바이더 정보들을 `integrations` 서비스에 바인딩할 때, 오직 환경변수(`env`) 매핑만 수행하고 **기본 인증 키 수집 방식인 `"key"` 메서드를 등록하지 않습니다**.
  * 이로 인해 커스텀 프로바이더의 `integration` 객체 내부의 `methods` 정보가 비어 있게 되며, SQLite 크레덴셜 정보가 없더라도 시스템은 무작정 통과시킵니다.

---

## 2. 해결 및 구현 방안 (Proposed Implementation)

커스텀 프로바이더도 인증 키가 저장되어 있지 않은 경우 사용자에게 AUTH KEY 입력을 요구하도록 강제하려면, 설정 플러그인이 커스텀 프로바이더를 가공할 때 **기본 인증 방법(`key` 타입)을 강제로 활성화**해야 합니다.

### 1) 소스코드 수정안 ([packages/core/src/config/plugin/provider.ts](file:///D:/AREA51/workspace/opencode/packages/core/src/config/plugin/provider.ts))

플러그인이 `integrations.update`를 처리할 때, 커스텀 프로바이더 통합 대상에 대해 항상 `method: { type: "key" }`를 설정해 주도록 코드를 보완합니다.

```diff
     yield* integrationTransform((integrations) => {
       for (const file of files) {
         for (const [id, item] of Object.entries(file.info.providers ?? {})) {
           const integrationID = Integration.ID.make(id)
           if (!configuredIntegrations.has(id) && !integrations.get(integrationID)) continue
           integrations.update(integrationID, (integration) => {
             integration.name = item.name ?? integration.name
           })
+          // 커스텀 프로바이더의 자격증명 등록을 활성화하기 위해 key 타입 인증 메서드 강제 등록
+          integrations.method.update({
+            integrationID,
+            method: { type: "key" },
+          })
           if (item.env !== undefined) {
             integrations.method.update({
               integrationID,
               method: { type: "env", names: [...item.env] },
             })
           }
         }
       }
     })
```

### 2) 설정 방식 규칙 (`opencode.json`)
인증 키 강제화를 위해서는 `opencode.json`에 커스텀 프로바이더를 추가할 때 `apiKey` 값을 직접 설정해 두지 않아야 합니다.

* **잘못된 예 (인증 프롬프트가 안 뜸)**:
  ```json
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "https://api.myprovider.com/v1",
        "apiKey": "direct-key-value-here" // 파일에 하드코딩되면 절대 묻지 않음
      }
    }
  }
  ```
* **올바른 예 (크레덴셜 연결 강제화)**:
  ```json
  "provider": {
    "myprovider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Custom AI Provider",
      "options": {
        "baseURL": "https://api.myprovider.com/v1"
      },
      "models": {
        "custom-model": {
          "name": "Custom Model"
        }
      }
    }
  }
  ```
  * 위와 같이 `apiKey`를 적어두지 않고, 환경변수 `env`도 만족하지 못할 경우 `catalog.ts`에서 `available` 판정이 `false`가 되어 UI 및 CLI 레벨에서 즉시 **AUTH KEY 입력 팝업**을 유도하게 됩니다.

---

## 3. 사용자 동의 및 동작 흐름 (User Connection Flow)

1. 사용자가 `myprovider`의 `custom-model`을 활성화하려고 시도합니다.
2. 시스템은 `available` 검사를 수행하고 `integration`이 로드되어 있으나 `connected`가 `false`이고 명시된 `apiKey`가 없으므로 해당 모델을 **연결 대기 상태**로 분류합니다.
3. UI는 자격증명 등록 모달([dialog-connect-provider.tsx](file:///D:/AREA51/workspace/opencode/packages/app/src/components/dialog-connect-provider.tsx))을 실행시킵니다.
4. 사용자가 API Key를 입력하면 `serverSDK().client.auth.set`을 통해 로컬 SQLite 데이터베이스에 암호화 저장됩니다.
5. 이후 대화 실행 시 `SessionRunnerModel.resolve`([packages/core/src/session/runner/model.ts](file:///D:/AREA51/workspace/opencode/packages/core/src/session/runner/model.ts#L157-L162))가 SQLite의 인증 데이터를 조회하여 `Authorization: Bearer <key>` 헤더를 주입해 요청을 안전하게 전송합니다.

---

## 4. 소스 코드 수정이 불가능한 경우의 대안 (Alternatives Without Core Modification)

OpenCode 코어 소스 코드를 빌드/수정할 수 없는 런타임 환경에서는 다음 대안들을 조합하여 동일한 사용성 및 안전성을 확보할 수 있습니다.

### 1) 어댑터(Adapter)를 이용한 마크다운 안내 메시지 스트리밍
OpenCode는 실행 도중 에러가 나더라도 동적으로 `/connect` 다이얼로그를 띄워주지 않습니다. 따라서 어댑터에 인증 값이 누락되어 유입되는 경우, 어댑터가 직접 `200 OK` 응답으로 가이드 문구를 스트리밍하는 것이 가장 직관적이고 효과적인 대안입니다.

* **동작 원리**:
  1. 어댑터가 수신한 HTTP 요청에서 `Authorization` 헤더를 검사합니다.
  2. 헤더가 누락되었거나 잘못된 경우, `200 OK`와 `Content-Type: text/event-stream`을 리턴합니다.
  3. `choices[0].delta.content` 필드에 설정 방법을 상세히 적어 SSE 스트림으로 사용자에게 즉시 뿌려줍니다.
  4. 스트림 출력이 끝난 후 `[DONE]` 신호로 정상 종료하여, OpenCode가 오류 종료 대신 대화창 안에서 사용자 안내 문구를 마크다운으로 렌더링하도록 합니다.

* **어댑터 스트리밍 데이터 예시**:
  ```text
  data: {"choices":[{"delta":{"content":"⚠️ **API 인증 키 설정이 필요합니다.**\n\n이 커스텀 모델을 사용하려면 API Key가 등록되어야 합니다.\n\nOpenCode 실행 환경에 아래와 같이 환경 변수를 선언하여 키를 공급해 주세요.\n\n**PowerShell:**\n```powershell\n$env:MY_PROVIDER_API_KEY=\"your-actual-api-key\"\n```\n\n**Bash:**\n```bash\nexport MY_PROVIDER_API_KEY=\"your-actual-api-key\"\n```"}}]}

  data: [DONE]
  ```

### 2) 안전한 API Key 동적 바인딩 설정 (`opencode.json`)
어시스턴트 안내를 통해 사용자가 터미널 환경 변수에 API Key를 설정하도록 유도한 경우, OpenCode는 소스 수정 없이 환경 변수 값을 어댑터에 주입할 수 있는 기능을 제공합니다.

#### 방법 A: `env` 환경 변수 리스트 바인딩 (권장)
`opencode.json` 내 커스텀 프로바이더 선언부에 `env` 설정을 추가합니다. OpenCode는 지정된 환경 변수가 런타임에 존재하면 이를 자동으로 조회하여 인증에 사용합니다.
```json
"provider": {
  "myprovider": {
    "npm": "@ai-sdk/openai-compatible",
    "name": "My Custom Provider",
    "env": ["MY_PROVIDER_API_KEY"],
    "options": {
      "baseURL": "https://api.myprovider.com/v1"
    },
    "models": {
      "custom-model": { "name": "Custom Model" }
    }
  }
}
```

#### 방법 B: `{env:VAR}` 치환 설정
문자열 옵션 내부에서 `{env:변수명}` 기법을 사용해 바인딩할 수도 있습니다.
```json
"options": {
  "baseURL": "https://api.myprovider.com/v1",
  "apiKey": "{env:MY_PROVIDER_API_KEY}"
}
```

### 3) 어댑터에서의 인증 토큰 검증 규격
OpenCode의 `@ai-sdk/openai-compatible` 러너는 자격 증명을 전송할 때 표준 헤더 포맷을 사용하여 인증을 시도합니다.
* **사용 헤더**: `Authorization`
* **헤더 구조**: `Bearer <등록한_API_KEY_값>`
* **검증 방식**: 어댑터는 유입되는 HTTP 요청에서 `req.headers['authorization']` 값을 가져온 후, `Bearer ` 문자열로 시작하는지 검증하고 그 뒷부분의 문자열을 사내/개인 인증 키와 매칭하여 처리하면 됩니다.

### 4) 자격 증명(API Key) 최종 저장 매커니즘
만약 향후 코어 수정을 통해 `/connect` 및 자격 증명 팝업을 활성화할 경우, 사용자가 화면상에서 입력하는 API Key의 동작 매커니즘은 다음과 같습니다.
* **보안 저장**: 입력된 인증 키는 평문 설정 파일(`opencode.json`)에 저장되지 않고, 로컬의 SQLite 암호화 보관소(`CredentialTable` 테이블)에 안전하게 격리 저장됩니다.
* **보관 경로**: OS별 데이터 홈 디렉토리 하위의 로컬 데이터베이스 파일에 기록됩니다.
  * Windows: `C:\Users\<사용자명>\.gemini\antigravity-cli\~` 또는 `AppData` 하위 경로의 SQLite DB 파일
  * macOS / Linux: `~/.config/opencode/` 또는 `~/.local/share/opencode/` 하위의 SQLite DB 파일
* **요청 결합**: API 호출 시 SQLite 로컬 세션 정보와 `SessionRunnerModel.resolve` 결합 프로세스가 활성화되어 암호화 저장소의 비밀키를 꺼내와 `Authorization: Bearer <key>` 헤더를 빌드하게 됩니다.

