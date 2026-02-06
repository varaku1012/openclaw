# OpenClaw Context for Gemini

## Project Overview
**OpenClaw** is a personal AI assistant designed to run on user devices. It acts as a central gateway, connecting various messaging channels (WhatsApp, Telegram, Slack, etc.) to AI agents. It supports local capabilities like voice wake, screen recording, and a "Canvas" for visual interactions.

*   **Type:** Monorepo (Node.js/TypeScript)
*   **Core Tech:** Node.js (>=22), TypeScript, pnpm, Docker, SQLite.
*   **Architecture:** Central "Gateway" (WebSocket control plane) managing sessions, channels, and tools.

## Development Environment
*   **Package Manager:** `pnpm` (v10.23.0 strict). Use `pnpm` for all script execution.
*   **Runtime:** Node.js >= 22.12.0.
*   **Monorepo:** Managed via `pnpm-workspace.yaml`.
    *   `src/`: Main application core (Gateway, CLI, Channels).
    *   `ui/`: Web frontend (Control UI, WebChat).
    *   `packages/`: Shared packages (e.g., `clawdbot`, `moltbot`).
    *   `extensions/`: Plugins for extra channels and features.
    *   `apps/`: Native companion apps (Android, iOS, macOS).

## Key Commands

### Setup & Build
*   **Install Dependencies:** `pnpm install`
*   **Build Core:** `pnpm build`
*   **Build UI:** `pnpm ui:build` (Required for the web interface)
*   **Full Build:** `pnpm prepack` (Builds core + UI)

### Running
*   **Dev Mode (Gateway):** `pnpm gateway:watch` (Auto-reloads on changes)
*   **Run CLI:** `pnpm openclaw <command>` (e.g., `pnpm openclaw onboard`)
*   **Start Gateway:** `pnpm start` (or `node scripts/run-node.mjs`)

### Testing & Quality
*   **Run Tests:** `pnpm test` (Runs parallel tests)
*   **Run All Tests:** `pnpm test:all` (Includes E2E, Docker, Lint)
*   **Lint:** `pnpm lint` (Uses `oxlint`)
*   **Format:** `pnpm format` (Uses `oxfmt`)
*   **Type Check:** `pnpm check` (Runs tsgo, lint, and format)

## Code Structure (`src/`)
The core logic resides in `src/` and is organized by domain:
*   `gateway/`: The WebSocket control plane and message routing.
*   `agents/`: AI agent logic and runtime.
*   `channels/`: Adapters for messaging platforms (WhatsApp, Telegram, Discord, etc.).
*   `sessions/`: User session management.
*   `nodes/` & `node-host/`: Device capability abstractions.
*   `tools/`: Agent tools (Browser, Canvas, etc.).
*   `cli/`: Command-line interface entry points.

## Development Conventions
*   **Style:** Adhere to existing formatting. Use `pnpm format:fix` to resolve style issues.
*   **Testing:** Writes tests for new features. Look at `src/**/*.test.ts` for examples.
*   **AI Contributions:** AI-generated code is explicitly welcomed (see `CONTRIBUTING.md`).
*   **Sandboxing:** New features should respect the project's security and sandboxing models (Docker-based execution for non-main sessions).

## Infrastructure
*   **Docker:** Used for sandboxing and deployment (`docker-compose.yml`, `Dockerfile`).
*   **Database:** SQLite with `sqlite-vec` for vector operations.
