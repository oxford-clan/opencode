### opencode 1.17.7 (dev@213ff3f2d) Analysis Documents

이 문서들은 `opencode` 1.17.7 로컬 리포지토리 기준으로, `opencode`와 `aiu-opencode-adapter` 연동에서 system prompt, tool calling, workflow 전달 계약이 실제로 어떻게 맞물리는지 검토하기 위해 작성한 로컬 분석 자료다.

- [opencode 소스 분석 리포트](./opencode_analysis/opencode-1.17.7/opencode-analysis-report.md) — opencode의 CLI, TUI, agent runtime, provider adapter, permission engine, session store, diff/review UI 구조를 소스 기준으로 정리한 전체 분석 문서.
- [opencode 분석 요약본](./opencode_analysis/opencode-1.17.7/opencode-analysis-summary.md) — opencode 소스 구조와 핵심 흐름을 빠르게 확인하기 위한 요약 문서.
- [aiu-opencode-adapter 프로젝트 분석](./opencode_analysis/opencode-1.17.7/aiu-opencode-adapter-analysis.md) — adapter 프로젝트의 전체 구조, 데이터 흐름, 구현 상태, 남은 리스크를 빠르게 파악하기 위한 종합 분석 문서. opencode 요청을 AIU Workflow로 중계하고 tool call을 에뮬레이션하는 전체 아키텍처를 설명한다.
- [opencode System Prompt 관리 방식 분석](./opencode_analysis/opencode-1.17.7/opencode-system-prompt-management-report.md) — opencode가 모델, agent, 요청 종류, 모드별로 system prompt를 조립하는 방식을 확인하기 위한 근거 문서. AGENTS.md/CLAUDE.md가 system prompt에 들어가는 경로와 adapter 추출 설계의 기준을 정리한다.
- [opencode `/init` AGENTS.md 생성 프롬프트 분석](./opencode_analysis/opencode-1.17.7/opencode-init-agents-md-prompt-report.md) — `/init`이 실제로 어떤 프롬프트를 user 메시지로 주입하고 AGENTS.md를 생성/갱신하는지 검증한 문서. 내장 agent 정의 파일과 `/init` command template의 역할 차이를 분리한다.
- [opencode System Prompt 교체 케이스 분석](./opencode_analysis/opencode-1.17.7/opencode-system-prompt-swap-cases.md) — `explore`, `title`, `summary`, `compaction` 등 agent별 base system prompt가 교체되는 케이스를 정리한 문서. `/init` 실행 중 `task` 도구가 explore subagent를 호출할 때 별도 child 세션 system prompt가 생기는 이유를 설명한다.
- [opencode `/init` 및 Agent별 System Prompt의 Workflow 전달 영향 검토](./opencode_analysis/opencode-1.17.7/opencode-init-workflow-system-prompt-impact-report.md) — `/init`과 subagent 실행 시 달라지는 system prompt가 현재 workflow에 전달되는지, 누락 시 영향도와 방지 방안을 정리한 의사결정용 검토 보고서. `opencode_instructions` 추가와 workflow pre-system context 변경 필요성을 다룬다.
- [aiu-opencode-adapter의 opencode agent 정보 식별 가능성 검토](./opencode_analysis/opencode-1.17.7/opencode-agent-info-adapter-report.md) — adapter가 opencode provider 요청만으로 현재 agent 이름을 알 수 있는지 확인하고, `x-opencode-agent` header 및 `opencode_agent` workflow input으로 명시 전달하는 방안을 정리한 보고서.
- [opencode Session 관리 정책 분석](./opencode_analysis/opencode-1.17.7/opencode-session-management-report.md) — opencode의 session ID 생성, DB 유지, HTTP path parameter 사용, provider outbound header/cache key 전달, `/new`/`/clear` 동작을 소스코드 레벨에서 정리한 문서.
- [opencode Context Usage 계산 방식 분석](./opencode_analysis/opencode-1.17.7/opencode-context-usage-report.md) — opencode가 API 응답 usage를 어떻게 토큰 필드로 정규화하고 context usage 퍼센트를 계산하는지 정리한 문서. aiu-opencode-adapter가 input messages 문자열 길이를 token 값으로 반환할 때 token 수는 갱신되지만 usage가 0%로 보일 수 있는 원인을 설명한다.
- [opencode Context 자동 압축 분석](./opencode_analysis/opencode-1.17.7/opencode-context-auto-compaction-report.md) — context usage 표시와 별개로 자동 compaction 임계값이 어떻게 계산되는지, `/compact` 명령이 어떤 marker와 hidden compaction agent 흐름을 실행하는지, `claude-opus-4-6` 100k/16k 설정에서 v1 기준 84%로 압축되는 이유를 정리한 보고서.
- [opencode 로그 및 Observability 구조 분석](./opencode_analysis/opencode-1.17.7/opencode-logging-observability-report.md) — opencode의 core/CLI 로그 파일, `--log-level`/`--print-logs`, OTLP remote 적재, desktop 로그 export 구조와 별도 경로 적재 가능성을 정리한 보고서.
- [opencode 권한 관리 정책 분석](./opencode_analysis/opencode-1.17.7/opencode-permission-management-report.md) — opencode가 어떻게 permission 권한을 규칙 및 와일드카드로 관리하고, opencode.json으로 설정할 수 있는지 상세 구조와 사용법을 정리한 보고서.
- [opencode `task` 권한 deny 영향 분석](./opencode_analysis/opencode-1.17.7/opencode-task-permission-deny-report.md) — `task` 권한을 deny했을 때 Task tool 노출, subagent 자동 위임, `/init`/`/review` 같은 slash command 실행 경로에 어떤 영향이 생기는지 소스코드 기준으로 정리한 보고서.
- [opencode Skill 과 Custom Tool 비교 분석](./opencode_analysis/opencode-1.17.7/opencode-skill-custom-tool-report.md) — opencode의 `SKILL.md` 기반 skill과 TypeScript/JavaScript custom tool의 목적, 로딩 경로, 권한 모델, 적용 방법, 장단점 및 조합 패턴을 비교 정리한 보고서.
- [opencode 단축키 및 커맨드 팔레트 개발자 가이드](./opencode_analysis/opencode-1.17.7/opencode-shortcuts-guide.md) — opencode의 단축키와 슬래시 커맨드, 커맨드 팔레트 키바인딩 표현 규칙 및 단축키 목록을 총망라하여 정리한 개발자 가이드 문서.
- [opencode 명령어 개발자 가이드](./opencode_analysis/opencode-1.17.7/opencode-commands-guide.md) — opencode CLI의 서브커맨드/옵션 별칭 구조 및 내장/사용자정의 슬래시 명령어 설정과 매칭 체계를 정리한 개발자 가이드 문서.
- [opencode Custom Provider Spec](./opencode_analysis/opencode-1.17.7/opencode-custom-provider-spec.md) — opencode custom provider 설정과 연결 흐름을 소스 및 문서 기준으로 정리한 스펙 문서.
- [opencode Custom Provider 가이드](./opencode_analysis/opencode-1.17.7/opencode-custom-provider-guide.md) — `/connect`, `opencode.json`, 코드 레벨 provider adapter 구현 기준을 단계별로 정리한 가이드 문서.
- [opencode 커스텀 프로바이더 인증 강제화 검토 보고서](./opencode_analysis/opencode-1.17.7/opencode-custom-provider-auth-spec.md) — 커스텀 프로바이더를 사용할 때 AUTH KEY 자격 증명 등록 과정을 강제하도록 하는 소스코드 레벨 구현 분석과 더불어, 코어 수정이 불가능한 환경을 위한 어댑터 가이드 스트리밍 및 환경 변수 바인딩 대안을 정리한 검토 보고서.

### opencode 1.17.9 (v1.17.9@5c23e8841) Analysis Documents

이 문서들은 현재 체크아웃된 `opencode` 1.17.9 기준으로, 기존 1.17.7 분석 결과와의 차이를 확인하기 위해 작성한 로컬 분석 자료다.

- [opencode 1.17.7 -> 1.17.9 System Prompt 변경 분석](./opencode_analysis/opencode-1.17.9/opencode-1.17.7-to-1.17.9-system-prompt-diff-report.md) — 1.17.7 분석 대비 1.17.9에서 agent별 system prompt 교체 구조, 실제 prompt txt 파일, max-steps prompt 이동, 후속 user message wrapping 제거 여부를 비교한 보고서.

### opencode 1.17.11 (dev@986846fbd) Analysis Documents

이 문서들은 현재 로컬 체크아웃된 `opencode` 1.17.11 기준으로, `REPORT.md`에 등록된 1.17.7/1.17.9 분석 결과와 최신 소스의 차이를 확인하기 위해 작성한 갭 분석 자료다.

- [opencode 1.17.9 -> 1.17.11 Gap Analysis](./opencode_analysis/opencode-1.17.11/opencode-1.17.9-to-1.17.11-gap-report.md) — 현재 1.17.11 소스 기준으로 기존 system prompt, agent 정보, instruction loading, context usage, compaction, task permission 분석이 유지되는지 검증하고, V2 Session Runner/SystemContext 및 GitLab workflow model 전용 `isWorkflow` 경로가 adapter 연동 분석에 만드는 갭을 정리한 보고서.
