# opencode 분석 요약본

## 한 줄 결론
opencode는 단순 CLI가 아니라, **터미널 중심의 AI 코딩 에이전트 플랫폼**이다. CLI, TUI, agent runtime, provider adapter, permission engine, session store, diff/review UI가 한 모노레포 안에 함께 들어 있다.

## 핵심 구조
- 진입점: [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1)
- 실행 흐름: [packages/opencode/src/cli/cmd/run.ts](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/run.ts#L1)
- TUI 루트: [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L221)
- 세션/런타임: [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L20)
- tool registry: [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L1)
- 세션 저장: [packages/core/src/session/sql.ts](D:/AREA51/workspace/opencode/packages/core/src/session/sql.ts#L117)

## 사용자 흐름
1. `opencode` 실행
2. config, provider, plugin, model 로딩
3. 세션 생성 또는 재개
4. prompt 입력 또는 slash command 실행
5. model streaming
6. tool 승인/실행
7. diff 및 응답 검토
8. 다음 turn 또는 세션 종료

## 가장 중요한 설계 포인트
- prompt admission과 model execution이 분리되어 있다.
- system context는 baseline/snapshot/revision 형태로 관리된다.
- tool 실행은 registry와 permission layer를 거친다.
- 파일 수정은 location boundary와 재검증을 통과해야 한다.
- markdown, diff, streaming output이 TUI에서 별도 렌더러로 처리된다.

## 강점
- 세션 재개와 durable history 관리가 강함
- permission / boundary / revalidation이 비교적 잘 설계됨
- TUI가 단순 텍스트 UI보다 풍부함
- plugin, provider, application tool로 확장 여지가 있음

## 리스크
- 구조가 크고 복잡해서 학습 곡선이 높음
- `bash`는 호스트 권한으로 실행되어 샌드박스가 강하지 않음
- `apply_patch`는 atomic rollback이 보장되지 않음
- provider catalog 범위와 실제 runtime adapter 범위가 다름
- 일부 경로는 TODO 또는 `확인 필요` 상태가 남아 있음

## 비교 관점
- Claude Code, Codex CLI: terminal agent UX는 유사하지만 opencode가 더 플랫폼형
- Aider: diff 중심은 비슷하지만 opencode는 agent runtime이 더 강함
- Cline: 승인 기반 assistant라는 점은 비슷하지만 IDE보다는 터미널 중심
- OpenHands: agent platform이라는 점은 유사하지만 opencode는 로컬 TUI/session 경험이 중심

## 학습 우선순위
1. [packages/opencode/src/index.ts](D:/AREA51/workspace/opencode/packages/opencode/src/index.ts#L1)
2. [packages/core/src/location-layer.ts](D:/AREA51/workspace/opencode/packages/core/src/location-layer.ts#L46)
3. [packages/core/src/session/runner/llm.ts](D:/AREA51/workspace/opencode/packages/core/src/session/runner/llm.ts#L82)
4. [packages/opencode/src/cli/cmd/tui/app.tsx](D:/AREA51/workspace/opencode/packages/opencode/src/cli/cmd/tui/app.tsx#L228)
5. [packages/core/src/tool/registry.ts](D:/AREA51/workspace/opencode/packages/core/src/tool/registry.ts#L108)

## 최종 판단
- opencode는 그대로 fork해서 쓰기보다, **구조를 이해하고 일부를 재사용하는 참고용 가치가 큰 코드베이스**다.
- 작은 CLI 에이전트를 만들기 위한 간결한 샘플은 아니고, 실제 제품형 agent platform에 가깝다.
