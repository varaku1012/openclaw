# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build, Test, and Development Commands

**Runtime:** Node 22+ required. Prefer `pnpm` for package management; Bun supported for TypeScript execution.

```bash
# Install dependencies
pnpm install

# Build (type-check + compile to dist/)
pnpm build

# Lint and format check
pnpm check

# Run all unit tests
pnpm test

# Run tests with coverage
pnpm test:coverage

# Run a single test file
pnpm vitest run src/path/to/file.test.ts

# Run tests in watch mode
pnpm test:watch

# Run CLI in development (TypeScript via tsx)
pnpm openclaw <command>

# Gateway development with auto-reload
pnpm gateway:watch

# E2E tests
pnpm test:e2e

# Live tests (requires real API keys)
OPENCLAW_LIVE_TEST=1 pnpm test:live
```

**Platform-specific:**
```bash
# macOS app packaging
pnpm mac:package

# iOS build/run
pnpm ios:build
pnpm ios:run

# Android build/run
pnpm android:run
```

## Architecture Overview

OpenClaw is a personal AI assistant with a **Gateway-centric architecture**:

```
Messaging Channels (WhatsApp/Telegram/Slack/Discord/Signal/iMessage/Teams/etc.)
                │
                ▼
┌───────────────────────────────────────┐
│              Gateway                   │
│     (WebSocket control plane)          │
│        ws://127.0.0.1:18789            │
└───────────────┬───────────────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
 Pi Agent    CLI Tools   Companion Apps
 (RPC mode)  (openclaw)  (macOS/iOS/Android)
```

### Key Modules

- **`src/gateway/`** — WebSocket server, session management, protocol handling, routing
- **`src/cli/`** — CLI wiring and command infrastructure
- **`src/commands/`** — Individual CLI commands (agent, gateway, send, etc.)
- **`src/channels/`** — Shared channel abstraction and routing logic
- **`src/agents/`** — Pi agent runtime integration, tool definitions
- **`src/providers/`** — Model provider implementations (Anthropic, OpenAI, etc.)
- **`src/media/`** — Media pipeline (images, audio, video processing)
- **`src/media-understanding/`** — Transcription and vision providers
- **`src/browser/`** — Browser control via Playwright/CDP
- **`src/sessions/`** — Session state management and persistence
- **`src/plugin-sdk/`** — SDK for building extension plugins

### Channel Implementations

**Core channels (in `src/`):** `telegram/`, `discord/`, `slack/`, `signal/`, `imessage/`, `whatsapp/`, `web/`, `line/`

**Extension channels (in `extensions/`):** `msteams/`, `matrix/`, `zalo/`, `zalouser/`, `bluebubbles/`, `googlechat/`, `voice-call/`, etc.

### Companion Apps

- **`apps/macos/`** — Swift menu bar app with Voice Wake, Canvas, gateway control
- **`apps/ios/`** — Swift iOS node with Canvas, camera, Voice Wake
- **`apps/android/`** — Kotlin Android node with Canvas, camera, Talk Mode
- **`apps/shared/`** — Shared OpenClawKit framework (Swift)

### Extensions/Plugins

Extensions live in `extensions/*/` as workspace packages. Keep plugin-only deps in the extension's `package.json`, not root. Avoid `workspace:*` in `dependencies` (breaks npm install); use `devDependencies` or `peerDependencies` instead.

## Coding Conventions

- **Language:** TypeScript (ESM). Use strict typing; avoid `any`.
- **Formatting:** Oxlint + Oxfmt. Run `pnpm check` before commits.
- **File size:** Keep files under ~500 LOC when feasible.
- **Tests:** Vitest with colocated `*.test.ts` files. Coverage threshold: 70%.
- **Naming:** Use **OpenClaw** in docs/UI; use `openclaw` for CLI/package/paths.
- **CLI progress:** Use `src/cli/progress.ts` (osc-progress + @clack/prompts spinner).
- **Terminal output:** Use palette from `src/terminal/palette.ts` (no hardcoded colors).

## Commit Guidelines

- Create commits with `scripts/committer "<msg>" <file...>` to keep staging scoped
- Use concise, action-oriented messages (e.g., `CLI: add verbose flag to send`)
- Changelog: keep latest version at top; add entry with PR # when merging external PRs
- Never commit real phone numbers, API keys, or live configuration values

## Key Configuration Files

- **`~/.openclaw/openclaw.json`** — Main user config (model, channels, agent settings)
- **`~/.openclaw/credentials/`** — Channel credentials (WhatsApp session, etc.)
- **`~/.openclaw/workspace/`** — Agent workspace with AGENTS.md, SOUL.md, skills/

## Important Patterns

- **Dependency injection:** Use `createDefaultDeps` pattern for testability
- **Tool schemas:** Avoid `Type.Union` in tool input schemas; use `stringEnum`/`optionalStringEnum` instead
- **SwiftUI (macOS/iOS):** Prefer `@Observable`/`@Bindable` over `ObservableObject`/`@StateObject`
- **Patched dependencies:** Any dep in `pnpm.patchedDependencies` must use exact version (no `^`/`~`)

## Testing

```bash
# Unit tests (fast, no network)
pnpm test

# E2E tests
pnpm test:e2e

# Live tests with real API keys
OPENCLAW_LIVE_TEST=1 pnpm test:live

# Docker-based tests
pnpm test:docker:all
```

Do not set test workers above 16. Pure test additions generally don't need changelog entries unless they alter user-facing behavior.

## Troubleshooting

- Run `openclaw doctor` to diagnose configuration and migration issues
- macOS logs: `./scripts/clawlog.sh` to query unified logs
- Gateway restart: use the OpenClaw Mac app or `scripts/restart-mac.sh`
