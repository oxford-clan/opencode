# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

OpenCode is an open-source AI coding agent built as a monorepo. The project includes a CLI tool, web UI, desktop application, admin console, and multiple supporting packages. It integrates with numerous LLM providers and provides both TUI and graphical interfaces for AI-powered development.

**Default branch**: dev (not main)
**Package manager**: Bun 1.3.14+
**Monorepo tool**: Turbo (orchestrates tasks across 25+ workspace packages)

### Related Analysis Documents

- [aiu-opencode-adapter 프로젝트 분석](./aiu-opencode-adapter-analysis.md) — opencode의 OpenAI-compatible 요청을 사내 AIU Workflow(chat-only LLM)로 중계하며 native tool call을 에뮬레이션하는 어댑터 프로젝트(`D:\AREA51\workspace\aiu-opencode-adapter`)의 구조·데이터 흐름·구현 상태 종합 분석.
- [opencode System Prompt 관리 방식 분석](./opencode-system-prompt-management-report.md) — opencode가 모델/agent/요청 종류/모드별로 system prompt를 구성하는 방식, 사전 정의 프롬프트(`session/prompt/*.txt`), AGENTS.md/CLAUDE.md 지시문 주입 경로 분석. aiu-opencode-adapter의 system prompt 추출/전송 설계 근거.
- [opencode `/init` AGENTS.md 생성 프롬프트 분석](./opencode-init-agents-md-prompt-report.md) — `/init`의 실제 프롬프트(`command/initialize.txt`)와 배선, `agent.ts`(내장 agent 정의 플러그인)와의 구분, AGENTS.md 작성 철학 및 adapter 적용점.
- [opencode System Prompt 교체 케이스 분석](./opencode-system-prompt-swap-cases.md) — system base prompt가 교체되는 단일 코드 지점(`request.ts:60`), subagent(`task`→`explore.txt`) 및 보조 호출(title/summary/compaction/generate) 카탈로그, `/init` 실행 시 explore subagent system 교체 경로 검증.

## Build, Test, and Lint Commands

### Installation and Development

```bash
# Install dependencies
bun install

# Start development server for main CLI (TUI)
bun dev

# Start dev servers for other targets
bun dev:desktop      # Electron desktop app
bun dev:web          # Web UI (SolidJS, packages/app)
bun dev:console      # Admin console app
bun dev:stats        # Analytics app
bun dev:storybook    # Component library
```

### Testing

```bash
# Run all tests across workspace via Turbo (generates JUnit reports)
bun turbo test:ci

# Run tests for a specific package
bun --cwd packages/opencode test
bun --cwd packages/app test:unit

# Run tests with watch mode
bun --cwd packages/app test:unit:watch

# Run e2e tests (Playwright)
bun --cwd packages/app test:e2e:local
bun --cwd packages/app test:e2e:ui      # Interactive UI mode

# Run HttpAPI exercise gates (coverage/auth/effect modes)
bun --cwd packages/opencode run test:httpapi
```

### Type Checking and Linting

```bash
# Type-check all packages (Turbo-coordinated)
bun typecheck

# Type-check a specific package
bun --cwd packages/opencode typecheck

# Lint code (oxlint with TypeScript awareness)
bun lint
```

### Building

```bash
# Build CLI executable (generates platform-specific binaries)
./packages/opencode/script/build.ts

# Build CLI with sourcemaps (for beta channel)
./packages/opencode/script/build.ts --sourcemaps

# Build standalone "localcode" executable
./packages/opencode/script/build.ts --single

# Build web/desktop (handled by respective dev servers or CI)
bun --cwd packages/app build
bun --cwd packages/desktop build
```

### Type Checking

Always run `bun typecheck` from package directories (e.g., `bun --cwd packages/opencode typecheck`), never `tsc` directly.

## Architecture

### Monorepo Structure

The workspace is organized into logical domains:

**Core packages**:
- packages/opencode – Main CLI, TUI (Terminal UI), and core orchestration logic
- packages/core – Low-level business logic: session management, LLM provider support, terminal execution, storage (SQLite/Drizzle ORM)

**UI & Frontend**:
- packages/app – Web UI components (SolidJS + Vite), shared between desktop and web
- packages/ui – Reusable UI component library with TailwindCSS
- packages/desktop – Electron wrapper around packages/app

**API & SDK**:
- packages/sdk/js – TypeScript SDK for programmatic access; generates code from OpenAPI schemas
- packages/server – Effect-based server utilities
- packages/plugin – Plugin system for extending OpenCode

**LLM & Providers**:
- packages/llm – Unified LLM provider abstraction; supports 15+ providers (OpenAI, Anthropic, Google, AWS Bedrock, etc.) via @ai-sdk/*

**Console (Admin Dashboard)**:
- packages/console/app – Frontend (SolidJS/Solid Start)
- packages/console/core – Business logic (Drizzle ORM, Stripe integration, PlanetScale/PostgreSQL)
- packages/console/mail – Email templates
- packages/console/resource – Resource management

**Stats & Analytics**:
- packages/stats/app – Analytics dashboard
- packages/stats/core – Data processing
- packages/stats/server – API server

**Documentation & Developer Tools**:
- packages/web – Landing page (Astro)
- packages/storybook – Component development and testing

**Utilities**:
- packages/effect-drizzle-sqlite – Effect-Drizzle SQLite integration layer
- packages/effect-sqlite-node – Node.js SQLite bindings for Effect
- packages/http-recorder – HTTP request/response recording utility
- packages/script – Shared build and utility scripts

### High-Level Data Flow

1. **CLI Entry Point** (packages/opencode/bin/opencode):
   - Dispatches to TUI (packages/opencode/src/cli/cmd/tui/), API server, or web interface
   - Built with Bun and compiled to platform-specific binaries

2. **Session Management** (packages/core/src/session/):
   - Manages multi-turn agent conversations
   - Persists state in SQLite via Drizzle ORM
   - Tracks context, tool results, and LLM responses

3. **LLM Provider Abstraction** (packages/llm/):
   - Unified interface over 15+ providers
   - Handles streaming, tool calling, token counting
   - Routes requests through @ai-sdk/* ecosystem

4. **Terminal Execution** (packages/core/src/pty/):
   - PTY management for shell commands (Bash, PowerShell)
   - Cross-platform via native bindings (@lydell/node-pty)

5. **TUI Rendering** (packages/opencode/src/cli/cmd/tui/):
   - SolidJS components rendered via OpenTUI terminal framework
   - Real-time updates, keyboard navigation, responsive layout

6. **Web & Desktop** (packages/app, packages/desktop):
   - Same SolidJS component library
   - Desktop: Electron wrapper with native integrations
   - Web: Browser-based, optionally connected to OpenCode API server

### Key Technologies

- **Effect** – Functional effect system for error handling, concurrency, and telemetry
- **Drizzle ORM** – Type-safe database layer with migrations
- **OpenTUI** – Terminal UI rendering framework (custom, from sst project)
- **SolidJS** – Reactive UI framework (fine-grained reactivity, no virtual DOM)
- **Vite** – Fast dev server and build tool
- **Electron** – Desktop application framework
- **Playwright** – E2E testing (Chrome only via browser automation)
- **Oxlint** – Fast TypeScript linter (Rust-based, with type awareness)
- **Bun** – JavaScript runtime and package manager (replaces Node.js)

### Database Schema

Managed via **Drizzle ORM** with migrations:
- SQLite (local, dev) or PlanetScale/PostgreSQL (cloud/console)
- Tables: sessions, context, artifacts, models, users, teams, billing (console)

```bash
# Generate/migrate database
bun --cwd packages/core run db -- migrate
bun --cwd packages/core run migration     # Custom migration helpers

# Console database management
bun --cwd packages/console/core run db           # Interactive shell
bun --cwd packages/console/core run db-dev       # Dev stage
bun --cwd packages/console/core run db-prod      # Production
```

### Configuration & Styling

- **TSConfig**: Extends @tsconfig/bun, located at root
- **Oxlint**: .oxlintrc.json with type-aware rules
- **TailwindCSS**: packages/ui/src/styles/tailwind/ for shared design tokens
- **Prettier**: Root-level config (120 char line width)
- **Bun Config**: bunfig.toml controls install behavior and test root

## Code Style and Patterns

Per AGENTS.md:

- **Functional style**: Use `flatMap`/`filter`/`map` with type guards over for loops
- **Inlining**: Inline values used only once; do not extract single-use helpers preemptively
- **Error handling**: Avoid `try`/`catch`; use Effect-based error handling
- **Type inference**: Rely on inference; avoid explicit annotations unless needed for exports
- **No `any`**: Avoid the `any` type
- **Destructuring**: Use dot notation (`obj.a`) rather than destructuring to preserve context
- **Variables**: Prefer `const`; use ternaries or early returns instead of reassignment
- **Control flow**: Avoid `else`; prefer early returns
- **Bun APIs**: Use `Bun.file()` and other Bun-native APIs over Node equivalents where available
- **In `src/config`**: Follow the self-export pattern at the top of each module (e.g., `export * as ConfigAgent from "./agent"`)

### Imports

- Never alias imports (`import { foo as bar }` is forbidden)
- Never use star imports (`import * as Foo from "..."`)
- Import namespace-style values by name: `import { Project } from "@opencode-ai/core/project"`, then use `Project.ID`
- Prefer dynamic imports for heavy modules only needed in specific code paths; destructure near the top of the narrowest scope

### Drizzle Schema

Use snake_case column field names so column names don't need string overrides:

```ts
// Good
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
})

// Bad
const table = sqliteTable("session", {
  projectID: text("project_id").notNull(),
})
```

### Testing

- Avoid mocks; test actual implementations
- Tests cannot run from repo root; run from package directories

### Commit Conventions

`type(scope): summary` — types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`; scopes: `core`, `opencode`, `tui`, `app`, `desktop`, `sdk`, `plugin`

## Important Notes

- **SDK Regeneration**: If you modify API/server code, run to regenerate the JavaScript SDK:
  ```bash
  ./packages/sdk/js/script/build.ts
  ```

- **No root tests**: The root bun test command intentionally fails; always run tests per-package or via bun turbo test:ci

- **Platform-specific builds**: CLI builds target Darwin, Linux (x64 & ARM), and Windows (x64 & ARM64) with native PTY modules

- **Agents**: OpenCode includes two built-in agents (switchable via Tab):
  - build – Full access agent (default)
  - plan – Read-only analysis agent
  - @general – Multi-step task subagent

## V2 Session Architecture Invariants

Key constraints for `packages/core/src/session/` and related code:

- `SessionV2.prompt(...)` admits one durable `session_input` row before scheduling advisory `SessionExecution.wake(sessionID)` unless `resume: false` (admit-only). Keep durable prompt admission separate from model execution.
- `SessionExecution` is process-global and Session-ID based — no layer should take a Session ID. Placement is discovered through `SessionStore` and `LocationServiceMap.get(session.location)`.
- `SessionRunner`, model resolution, tool registry, permissions, and filesystem are Location-scoped. Omitted `Location.workspaceID` means implicit-local.
- One explicit `llm.stream(request)` call per provider turn; reload projected history before durable continuation.
- `SessionRunCoordinator` joins explicit same-Session resumes, coalesces prompt wakeups, and allows different Sessions concurrently. Advisory wakes drain eligible durable inbox rows only.
- Prompts steer by default and coalesce into the active activity. Explicit `queue` inputs open FIFO future activities after the active activity settles.

## Running OpenCode Locally

### Development vs. Production Equivalents

During development, bun dev is the local equivalent of the built opencode command:

```bash
# Development (from project root)
bun dev --help                # Show all available commands
bun dev serve                 # Start headless API server (port 4096)
bun dev web                   # Start server + open web interface
bun dev <directory>           # Start TUI in specific directory
bun dev spawn                 # Run TUI with server in spawned process (useful for debugging)
bun dev attach <url>          # Attach TUI to running server

# Running against different directories
bun dev .                      # Run OpenCode on opencode repo itself
bun dev /path/to/project      # Run on any other project
```

### Web App Development

To test UI changes:

1. Start the OpenCode server: bun dev serve
2. In another terminal: bun --cwd packages/app dev
3. Opens at http://localhost:5173 (or similar)

### Desktop App Development

```bash
bun --cwd packages/desktop dev          # Dev mode
bun --cwd packages/desktop build        # Production build
bun --cwd packages/desktop package      # Create installer
```

## Debugging

- Most reliable method: bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve
- For TUI debugging with server breakpoints: Use bun dev spawn or debug server separately and attach TUI
- VSCode: Use example configs in .vscode/launch.example.json and .vscode/settings.example.json
- Logs: Check ~/.opencode/logs for local CLI output

## Pull Request Guidelines

- **Issue First**: All PRs must reference an existing issue (use Fixes #123 or Closes #123)
- **Small & Focused**: Keep PRs focused on a single change
- **UI Changes**: Include screenshots or videos of before/after
- **Logic Changes**: Explain how you verified the fix/feature works
- **Style Guide**: Follow AGENTS.md conventions

## Common Development Patterns

### Running a Single Test

```bash
# Unit test
bun --cwd packages/TARGET test -- --testNamePattern="pattern"

# E2E test
bun --cwd packages/app test:e2e:local -- --grep="pattern"
```

### Adding a New Package

1. Create packages/new-package/package.json with name and scripts
2. Use catalog versions for dependencies: "effect": "catalog:"
3. Export from src/index.ts
4. Packages are auto-discovered; no manual registration needed

### Adding an LLM Provider

1. Providers are configured in packages/core/src/config/
2. Model definitions are in packages/core/src/config/model.ts
3. For custom providers, implement via @ai-sdk/provider interface
4. Update models.dev repository for new providers