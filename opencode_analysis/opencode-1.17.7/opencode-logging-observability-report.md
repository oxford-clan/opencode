# opencode 로그 및 Observability 구조 분석

> 검토 대상: `D:\AREA51\workspace\opencode`
> 작성 일자: 2026-06-20
> 목적: opencode가 자체 로그를 남기는지, 로그 저장 경로와 level 변경 방식이 무엇인지, 다른 경로나 remote로 적재 가능한지 소스코드 기준으로 정리한다.

---

## 1. 결론

opencode는 자체 로그를 남긴다. 현재 구조는 크게 세 갈래다.

| 구분 | 저장/전송 방식 | 주요 코드 |
| --- | --- | --- |
| core/CLI/server 로그 | XDG data 아래 `opencode/log/opencode.log` 파일 append | `packages/core/src/observability/logging.ts` |
| stderr 출력 | `--print-logs` 또는 `OPENCODE_PRINT_LOGS=1`일 때 파일 로그에 추가 | `packages/core/src/observability/logging.ts`, `packages/opencode/src/index.ts` |
| remote observability | `OTEL_EXPORTER_OTLP_ENDPOINT`가 있으면 OTLP logs/traces 전송 | `packages/core/src/observability/otlp.ts` |
| desktop 앱 로그 | Electron `userData/logs/<timestamp>/` 아래 scope별 파일 | `packages/desktop/src/main/logging.ts` |

CLI/core 로그의 level은 `--log-level DEBUG|INFO|WARN|ERROR` 또는 `OPENCODE_LOG_LEVEL`로 바뀐다. 기본값은 `INFO`다.

로그 파일 경로만 별도로 바꾸는 공식 CLI 옵션은 현재 보이지 않는다. 파일 로그 경로를 옮기려면 `XDG_DATA_HOME`으로 opencode data root 전체를 바꾸거나, `--print-logs`로 stderr에 출력한 뒤 외부 로그 수집기/리다이렉션으로 보내는 방식이 현실적이다.

remote 적재는 표준 OTLP 환경변수로 가능하다. `OTEL_EXPORTER_OTLP_ENDPOINT`를 설정하면 logs는 `${endpoint}/v1/logs`, traces는 `${endpoint}/v1/traces`로 전송된다.

---

## 2. Core/CLI 로그 저장 방식

core logging은 Effect logger 기반이다. 모든 주요 Effect runtime은 `Observability.layer`를 제공받는다.

대표 연결 지점:

- `packages/core/src/effect/runtime.ts`
- `packages/opencode/src/effect/app-runtime.ts`
- `packages/opencode/src/effect/bootstrap-runtime.ts`
- `packages/opencode/src/effect/run-service.ts`
- `packages/opencode/src/server/routes/instance/httpapi/server.ts`

`Observability.layer`는 `packages/core/src/observability.ts:12-19`에서 logger layer와 tracing layer를 병합한다.

```ts
const logs = Logger.layer([...Logging.loggers(), ...Otlp.loggers()], { mergeWithExisting: false }).pipe(
  Layer.provide(NodeFileSystem.layer),
  Layer.provide(OtlpSerialization.layerJson),
  Layer.provide(FetchHttpClient.layer),
  Layer.orDie,
  Layer.merge(Layer.succeed(References.MinimumLogLevel, Logging.minimumLogLevel())),
)
return Layer.merge(logs, yield* Effect.promise(Otlp.tracingLayer))
```

기본 파일 logger는 `packages/core/src/observability/logging.ts:49-51`에 있다.

```ts
export function fileLogger(file = path.join(Global.Path.log, "opencode.log"), id: string = runID) {
  return Logger.toFile(formatter(id), file, { flag: "a" })
}
```

즉 기본 파일명은 `opencode.log`이고, 같은 파일에 append한다.

---

## 3. 로그 디렉터리 산정

로그 루트는 `packages/core/src/global.ts`에서 정한다.

```ts
const data = path.join(xdgData!, app)

const paths = {
  data,
  log: path.join(data, "log"),
  ...
}
```

문서상 기본 경로:

| OS | 경로 |
| --- | --- |
| macOS/Linux | `~/.local/share/opencode/log/` |
| Windows | `%USERPROFILE%\.local\share\opencode\log` |

코드상으로는 `xdg-basedir`의 `xdgData`를 쓰므로 `XDG_DATA_HOME`을 설정하면 data root와 함께 로그 경로도 이동한다.

주의: 이는 로그만 옮기는 옵션이 아니라 opencode data root 전체에 영향을 주는 방식이다.

---

## 4. 로그 포맷

`packages/core/src/observability/logging.ts`의 formatter는 Effect structured log를 key-value 한 줄 문자열로 바꾼다.

포함 필드:

- `timestamp`
- `level`
- `run`
- message
- cause
- spans
- annotations

nested object는 flatten된다. 예를 들어 `{ request: { timing: { duration: 42 } } }`는 `request.timing.duration=42` 형태가 된다.

테스트도 이 동작을 검증한다.

- `packages/core/test/effect/observability.test.ts`
  - concurrent run이 같은 `opencode.log`에 append될 때 각 줄에 `run=<id>`가 붙는지 확인
  - nested object flatten 확인

---

## 5. 로그 level 변경 방식

CLI entrypoint는 `packages/opencode/src/index.ts:53-68`에서 전역 옵션을 받는다.

```ts
.option("print-logs", {
  describe: "print logs to stderr",
  type: "boolean",
})
.option("log-level", {
  describe: "log level",
  type: "string",
  choices: ["DEBUG", "INFO", "WARN", "ERROR"],
})
.middleware(async (opts) => {
  if (opts.printLogs) process.env.OPENCODE_PRINT_LOGS = "1"
  if (opts.logLevel) process.env.OPENCODE_LOG_LEVEL = opts.logLevel
})
```

실제 level 결정은 `packages/core/src/observability/logging.ts:56-64`에서 한다.

```ts
export function minimumLogLevel() {
  const value = process.env.OPENCODE_LOG_LEVEL?.toUpperCase()
  const levels = {
    DEBUG: "Debug",
    INFO: "Info",
    WARN: "Warn",
    ERROR: "Error",
  } as const satisfies Record<string, LogLevel.LogLevel>
  return value && value in levels ? levels[value as keyof typeof levels] : levels.INFO
}
```

정리:

| 방법 | 동작 |
| --- | --- |
| `opencode --log-level DEBUG` | `OPENCODE_LOG_LEVEL=DEBUG` 설정 후 minimum level을 Debug로 변경 |
| `OPENCODE_LOG_LEVEL=WARN opencode ...` | env에서 직접 minimum level 변경 |
| 값 없음/잘못된 값 | 기본 `INFO` |

---

## 6. stderr 출력

`--print-logs`는 파일 로그를 대체하지 않는다. 파일 logger에 stderr logger를 추가한다.

`packages/core/src/observability/logging.ts:67-69`:

```ts
export function loggers() {
  return process.env.OPENCODE_PRINT_LOGS === "1" ? [fileLogger(), stderrLogger] : [fileLogger()]
}
```

따라서 다음처럼 쓰면 파일 로그와 터미널 출력이 동시에 생긴다.

```sh
opencode --print-logs --log-level DEBUG
```

stderr를 외부 로그 수집기로 넘기려면 shell redirection, systemd journal, Docker logging driver, sidecar log collector 같은 실행 환경 레벨의 수집기를 붙이면 된다.

---

## 7. config `logLevel`의 현재 상태

legacy v1 config schema에는 `logLevel` 필드가 남아 있다.

`packages/core/src/v1/config/config.ts:27-37`:

```ts
const LogLevelRef = Schema.Literals(["DEBUG", "INFO", "WARN", "ERROR"])

export const Info = Schema.Struct({
  logLevel: Schema.optional(LogLevelRef).annotate({ description: "Log level" }),
  ...
})
```

그러나 v2 config review에서는 이 필드를 port하지 않는 것으로 정리되어 있다.

`specs/v2/config.md:31`:

```text
logLevel | Intended logging level configuration | remove | Do not port: no config consumer exists and logging initializes from CLI input.
```

즉 현재 런타임 관점에서 신뢰할 수 있는 level 변경 경로는 CLI 옵션 또는 env다.

SDK의 `createOpencodeServer()`는 `options.config?.logLevel`이 있으면 내부적으로 `--log-level=...` 인자를 붙인다. config 파일을 runtime consumer가 직접 읽는 구조가 아니라 SDK가 CLI option으로 변환하는 구조다.

---

## 8. Remote 적재: OTLP

remote logs/traces는 OTLP 환경변수로 켠다.

`packages/core/src/observability/otlp.ts:7-18`:

```ts
const endpoint = Flag.OTEL_EXPORTER_OTLP_ENDPOINT

const headers = Flag.OTEL_EXPORTER_OTLP_HEADERS
  ? Flag.OTEL_EXPORTER_OTLP_HEADERS.split(",").reduce(...)
  : undefined
```

logs exporter:

```ts
export function loggers() {
  if (!endpoint) return []
  return [OtlpLogger.make({ url: `${endpoint}/v1/logs`, resource: resource(), headers })]
}
```

traces exporter:

```ts
new OTLP.OTLPTraceExporter({
  url: `${endpoint}/v1/traces`,
  headers,
})
```

사용 예:

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%20token \
OTEL_RESOURCE_ATTRIBUTES=service.namespace=local,team=aiu \
opencode --log-level INFO
```

리소스 속성에는 다음 built-in 값이 들어간다.

- `serviceName: "opencode"`
- `serviceVersion: InstallationVersion`
- `deployment.environment.name`
- `opencode.client`
- `opencode.run`
- `service.instance.id`

`OTEL_RESOURCE_ATTRIBUTES`가 같은 key를 제공해도 built-in `opencode.client`, `service.instance.id` 등은 마지막에 덮어써지는 구조다.

workspace child process에도 OTLP env가 전달된다.

`packages/opencode/src/control-plane/workspace.ts:545-551`:

```ts
const env = {
  OPENCODE_AUTH_CONTENT: JSON.stringify(yield* auth.all()),
  OPENCODE_WORKSPACE_ID: config.id,
  OPENCODE_EXPERIMENTAL_WORKSPACES: "true",
  OTEL_EXPORTER_OTLP_HEADERS: process.env.OTEL_EXPORTER_OTLP_HEADERS,
  OTEL_EXPORTER_OTLP_ENDPOINT: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  OTEL_RESOURCE_ATTRIBUTES: process.env.OTEL_RESOURCE_ATTRIBUTES,
}
```

---

## 9. Plugin 로그

plugin은 `console.log` 대신 `client.app.log()`를 써서 opencode 서버 로그에 structured log를 남길 수 있다.

문서 예시: `packages/web/src/content/docs/ko/plugins.mdx:318-331`

```ts
await client.app.log({
  body: {
    service: "my-plugin",
    level: "info",
    message: "Plugin initialized",
    extra: { foo: "bar" },
  },
})
```

서버 handler는 `packages/opencode/src/server/routes/instance/httpapi/handlers/control.ts:24-34`에서 level에 따라 Effect logger를 호출한다.

```ts
const write =
  ctx.payload.level === "debug"
    ? Effect.logDebug
    : ctx.payload.level === "info"
      ? Effect.logInfo
      : ctx.payload.level === "warn"
        ? Effect.logWarning
        : Effect.logError
yield* write(ctx.payload.message).pipe(Effect.annotateLogs(ctx.payload.extra ?? {}))
```

따라서 plugin log도 core log sink를 공유한다. 파일, stderr, OTLP 설정이 그대로 적용된다.

---

## 10. Desktop 앱 로그

Desktop 앱은 Electron main process에서 별도 로그 체계를 쓴다.

`packages/desktop/src/main/logging.ts`:

- `root = join(app.getPath("userData"), "logs")`
- `run = join(root, stamp())`
- 파일 크기 제한: `log.transports.file.maxSize = 5 * 1024 * 1024`
- run 디렉터리 안에 scope별 로그 파일 생성
- 7일이 지난 로그 디렉터리 cleanup
- network log: `network.netlog`, 최대 20MB
- crash dump: `app.getPath("userData")/Crashpad`

로그 파일명은 scope 기반이다.

```ts
log.transports.file.resolvePathFn = (_vars, message) =>
  join(
    run,
    `${safeLogName(message?.scope ?? (message?.variables?.processType === "renderer" ? "renderer" : "main"))}.log`,
  )
```

Desktop은 `Export Logs...` 메뉴/command를 제공한다.

- `packages/app/src/desktop-menu.ts`
- `packages/app/src/pages/layout.tsx`
- `packages/desktop/src/main/logging.ts:51-75`

`exportDebugLogs()`는 다음을 zip으로 묶어 Downloads에 저장한다.

- desktop run logs
- server log roots
- Crashpad dump
- manifest
- 최근 24시간 내, 파일당 50MB 이하
- `.heapsnapshot`은 제외

---

## 11. Desktop sidecar와 server 로그

Desktop 앱은 background에서 opencode server sidecar를 실행한다.

local sidecar는 desktop process env를 넘겨받는다. `preferAppEnv()`는 desktop 실행 환경에 다음 값을 설정한다.

- `OPENCODE_CLIENT=desktop`
- `XDG_STATE_HOME`
- filewatcher/icon discovery 관련 env

WSL sidecar는 별도 script에서 명시적으로 다음과 같이 실행된다.

`packages/desktop/src/main/wsl/sidecar.ts:29-37`:

```sh
exec opencode --print-logs --log-level WARN serve --hostname 0.0.0.0 --port ...
```

개발 빌드에서는 `INFO`, packaged 앱에서는 `WARN`을 사용한다.

WSL sidecar의 stdout/stderr는 desktop logger로 회수된다. 따라서 WSL server 로그는 WSL 내부 파일 로그에도 남고, `--print-logs` 경유로 desktop 로그에도 들어간다.

---

## 12. 다른 로컬 경로 적재 가능성

현재 확인된 공식 CLI 옵션에는 로그 파일 경로를 직접 지정하는 옵션이 없다.

가능한 방식:

| 방식 | 설명 | 주의 |
| --- | --- | --- |
| `XDG_DATA_HOME` 변경 | `Global.Path.log`가 data root 기반이므로 로그 경로도 이동 | 로그만 아니라 opencode data root 전체가 이동 |
| `--print-logs` + redirection | stderr 출력으로 외부 파일/수집기에 전달 | 파일 로그는 계속 남음 |
| systemd/Docker/프로세스 매니저 수집 | stderr/stdout 기반 운영 로그 수집 | opencode 내부 파일 경로 변경은 아님 |
| 코드 변경 | `fileLogger(file)`는 임의 파일 인자를 받을 수 있음 | public CLI/config로 노출되어 있지는 않음 |

예:

```sh
XDG_DATA_HOME=/var/lib/opencode-data opencode serve
```

```sh
opencode --print-logs --log-level DEBUG 2>> /var/log/opencode/opencode.stderr.log
```

---

## 13. 부가 진단 파일

일반 로그 외에 다음 진단 파일 경로도 있다.

### Heap snapshot

`OPENCODE_AUTO_HEAP_SNAPSHOT=1`이면 RSS가 2GB를 넘을 때 `Global.Path.log` 아래에 heap snapshot을 쓴다.

`packages/opencode/src/cli/heap.ts:13-31`:

```ts
const file = path.join(
  Global.Path.log,
  `heap-${process.pid}-${new Date().toISOString().replace(/[:.]/g, "")}.heapsnapshot`,
)
```

### Direct trace

`packages/opencode/src/cli/cmd/run/trace.ts`에는 dev-only JSONL trace가 정의되어 있다.

```text
OPENCODE_DIRECT_TRACE=1
~/.local/share/opencode/log/direct/<timestamp>-<pid>.jsonl
~/.local/share/opencode/log/direct/latest.json
```

다만 현재 검색 기준으로는 `trace()` 호출 지점이 보이지 않는다. 따라서 일반 사용 경로의 활성 기능이라기보다 남아 있는 개발용 진단 코드로 보는 것이 맞다.

---

## 14. 문서와 코드의 차이

`packages/web/src/content/docs/troubleshooting.mdx`와 한국어 문서는 로그 파일이 timestamp 파일이고 최근 10개가 보관된다고 설명한다.

하지만 현재 core logging code는 `Global.Path.log/opencode.log` 단일 파일에 append한다. 최근 10개 rotation 로직도 core logger 쪽에서는 확인되지 않았다.

Desktop 로그는 run별 timestamp 디렉터리를 만들고 7일 cleanup을 수행하므로, 문서의 timestamp 설명은 desktop 또는 과거 구현과 섞여 있을 가능성이 있다.

---

## 15. 운영 판단

일반 CLI/server 운영에서 가장 직접적인 설정은 다음이다.

```sh
opencode --log-level DEBUG --print-logs
```

파일 위치:

```text
~/.local/share/opencode/log/opencode.log
```

remote observability를 붙이려면 다음 env를 쓴다.

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%20...
OTEL_RESOURCE_ATTRIBUTES=service.namespace=aiu,env=local
```

로그 파일만 별도 경로로 보내야 한다면 현재는 공식 옵션이 없으므로 `XDG_DATA_HOME` 변경 또는 stderr 수집이 안전하다. code-level 변경을 허용한다면 `fileLogger()`의 파일 경로를 설정 가능한 env/CLI option으로 노출하는 방식이 가장 작다.
