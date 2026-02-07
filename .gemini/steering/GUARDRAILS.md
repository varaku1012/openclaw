# Engineering Guardrails & Standards

This document defines the non-negotiable rules, quality standards, and development workflows for AI coding agents working on OpenClaw.

## 1. Environment & Key Commands

- **Runtime:** Node.js >= 22.12.0, pnpm v10.23.0
- **Build:** `pnpm build` (core), `pnpm ui:build` (frontend)
- **Dev:** `pnpm dev` (hot-reload CLI/Gateway)
- **Quality Gate (Run before every commit):** `pnpm check` (Runs lint, format, and type-check)
- **Test:** `pnpm test` (unit), `pnpm test:e2e` (end-to-end)

## 2. Non-Negotiables (Never Violate)

- **Generated Output:** Do not edit files under `dist/` directly. Always modify source in `src/`.
- **Config Changes:** Do not bypass schema validation. Extend `src/config/types*.ts` and the corresponding Zod schema files.
- **CLI Commands:** Do not hardcode commands. Wire them through the CLI registrars in `src/cli/program/`.
- **Channel Abstractions:** Do not import heavy channel modules into shared paths. Use the `ChannelDock` or `ChannelRegistry` abstractions (`src/channels/dock.ts`, `src/channels/registry.ts`).
- **Session Keys:** Never change the session key derivation logic without extreme caution. Session keys MUST be deterministic (`src/routing/session-key.ts`).
- **Security:** Gateway WS handshake must pass auth/role checks (`src/gateway/server/ws-connection/message-handler.ts`).
- **Authorization:** Always check scopes before handler execution in `authorizeGatewayMethod()`.
- **Sandbox Safety:** Paths must be validated via `assertSandboxPath()` to prevent traversal.

## 3. Architectural Patterns

### Protocol & RPC
- Define schema in `src/gateway/protocol/schema/`.
- Register validation exports in `src/gateway/protocol/index.ts`.
- Wire the handler in `src/gateway/server-methods/`.
- Run `pnpm protocol:gen` after changes.

### Agent Execution
- Core loop: `src/agents/pi-embedded-runner.ts`.
- Session resolution: `src/commands/agent/session.ts`.
- Model normalization: `src/agents/model-selection.ts`.

### Outbound Messaging
- Route all sends through outbound adapters and `src/infra/outbound/deliver.ts`.
- Do not add channel-specific branching in core modules.

## 4. Resilience Standards (Write Robust Code)

- **Exponential Backoff:** Use it for all external API calls.
- **Circuit Breakers:** Implement for critical integrations to "fail fast" during outages.
- **Timeouts:** Always set explicit, aggressive timeouts (5-30s for APIs, 60s+ for media).
- **Graceful Degradation:** Failures in non-critical components (like vision/transcription) should not crash the main message flow.
- **Atomic Writes:** When writing to disk (sessions/configs), use temp-file-and-rename patterns.

## 5. Testing Requirements

- **Colocation:** Tests MUST be colocated: `filename.ts` -> `filename.test.ts`.
- **E2E:** Critical gateway/agent flows require `*.e2e.test.ts`.
- **Validation:** Every feature must be validated by the full build and test suite.

## 6. Done Criteria

A task is complete ONLY when:
1. Code is wired through canonical seams (not shortcuts).
2. Config/protocol/schema updates are complete and validated.
3. Relevant tests are added or updated.
4. `pnpm check` passes with zero issues.
5. At least one execution path is validated (via test or command run).