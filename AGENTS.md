# opencode Analysis Workspace

이 저장소는 upstream open-source `opencode` 소스코드를 분석해 사내 사용 및 `aiu-opencode-adapter` 연동 판단에 필요한 근거를 정리하는 작업 공간이다.

## Role

- 기본 역할은 `opencode` 소스코드 분석, 동작 검증, 분석 문서 작성이다.
- `opencode` 소스코드를 수정하지 않는다. 사용자가 명시적으로 코드 변경을 요청한 경우에만 예외로 한다.
- `D:\AREA51\workspace\aiu-opencode-adapter` 프로젝트는 수정하지 않는다. 해당 프로젝트는 현재 분석의 참고 대상이며, 읽기 전용으로만 사용한다.
- adapter 관련 결론이나 권고는 이 저장소의 분석 문서 또는 보고서로 작성한다. 사용자가 명시적으로 adapter 프로젝트 작업을 요청하기 전까지 adapter 파일을 변경하지 않는다.

## Primary Sources

- `REPORT.md`는 지금까지 작성한 분석 문서의 색인이다. 새 분석을 시작하기 전에 먼저 확인한다.
- 상세 분석 문서는 `opencode_analysis/` 아래에 있다.
- adapter 프로젝트의 현재 구현 상태가 필요한 경우 `D:\AREA51\workspace\aiu-opencode-adapter`를 읽어 확인하되, 편집하지 않는다.
- upstream `opencode` 동작은 실제 소스 파일을 근거로 확인한다. 추정으로 결론을 쓰지 말고 관련 파일 경로를 분석 문서에 남긴다.

## Analysis Priorities

- opencode가 provider request를 구성하는 방식.
- system prompt, agent prompt, `AGENTS.md`/`CLAUDE.md` instruction 주입 방식.
- session, message history, tool call, subagent, permission, compaction 흐름.
- custom provider, OpenAI-compatible API, streaming SSE, usage/context 계산 방식.
- `aiu-opencode-adapter`와 맞물리는 요청/응답 계약 및 drift 위험.
- 버전 변경 시 기존 분석 결론이 유지되는지 여부.

## Output Guidelines

- 분석 문서는 재현 가능한 근거 중심으로 작성한다.
- 결론을 먼저 쓰고, 그 뒤에 근거 파일과 코드 흐름을 정리한다.
- 한국어 설명을 기본으로 하되, API 필드명, 파일 경로, 타입명, 함수명은 원문 그대로 쓴다.
- adapter에 필요한 조치가 보이면 "adapter 수정"을 직접 수행하지 말고, 요구사항/리스크/권장 설계로 정리한다.
- 기존 분석과 중복되는 내용은 `REPORT.md`의 기존 문서를 먼저 참조하고, 새로 확인한 차이나 보완점만 추가한다.

## Editing Rules

- 기본적으로 분석 문서, 보고서, 색인 문서만 수정한다.
- `packages/`, `sdks/`, `infra/` 등 upstream 소스 영역은 읽기 전용으로 취급한다.
- 사용자가 명시적으로 소스 변경을 요청한 경우에는 변경 전에 범위와 영향 파일을 분명히 확인한다.
- 새 분석 문서를 추가하면 `REPORT.md`에 링크와 한 줄 목적을 추가한다.

## Verification

- 분석 결론은 가능한 한 `rg`, 파일 열람, diff 확인 등 로컬 소스 근거로 검증한다.
- opencode 버전 차이를 다룰 때는 대상 버전, branch 또는 commit 기준을 문서에 명시한다.
- 실행 검증이 필요한 경우 실제 실행 명령과 관찰 결과를 문서에 남긴다.
- 테스트나 빌드는 분석 근거 확보에 필요할 때만 실행한다.
