# Engineering Guardrails

Last validated: 2026-02-06

This document defines implementation rules for AI coding agents changing OpenClaw.

## 1) Non-Negotiables

- Do not edit generated output under `dist/` directly.
- Do not bypass schema validation by adding ad-hoc config reads; extend `src/config/types*.ts` + Zod schema files.
- Do not introduce new command families by hardcoding in random files; wire through CLI registrars (`src/cli/program/*`).
- Do not bypass channel/plugin contracts by importing heavy channel modules into shared paths; use channel dock/registry abstractions first (`src/channels/dock.ts`, `src/channels/registry.ts`).
- Keep plugin-facing APIs in `src/plugin-sdk/index.ts` stable and additive.

## 2) Security-Sensitive Invariants

- Gateway WS handshake must begin with `connect` and pass auth/role checks (`src/gateway/server/ws-connection/message-handler.ts`).
- Gateway method authorization scope model in `src/gateway/server-methods.ts` must remain coherent when adding methods.
- Node command execution must pass allowlist checks (`src/gateway/node-command-policy.ts`, `src/gateway/server-methods/nodes.ts`).
- DM/group access control must stay explicit (`dmPolicy`, allowlists, group mention gating).
- Hook/webhook auth token checks in gateway HTTP path must remain mandatory when hooks are enabled (`src/gateway/server-http.ts`, `src/gateway/hooks.ts`).

## 3) Architectural Patterns to Follow

### CLI

- Add or change top-level command behavior via `src/cli/program/command-registry.ts` and sub-registrars.
- For new subcommands, add lazy registration entries in `src/cli/program/register.subclis.ts`.

### Agent execution

- Start from `src/commands/agent.ts` and shared helper modules under `src/commands/agent/`.
- Keep session resolution logic centralized (`src/commands/agent/session.ts`).
- Keep model/provider normalization in `src/agents/model-selection.ts` and related catalog/fallback files.

### Outbound messaging

- Route all channel send behavior through outbound adapters and `src/infra/outbound/deliver.ts`.
- Avoid adding channel-specific branching in unrelated core modules.

### Plugins/extensions

- New extension capability should register through plugin API, not by patching gateway core first.
- Plugin manifests (`extensions/<extension-id>/openclaw.plugin.json`) must include valid `configSchema`.

## 4) Type and Schema Rules

- TypeScript strict typing only; avoid `any`.
- For config additions:
  - Add types under `src/config/types.*.ts`
  - Add schema under `src/config/zod-schema*.ts`
  - Ensure `config.schema` and validation paths continue to work.
- For protocol additions:
  - Add TypeBox schema under `src/gateway/protocol/schema/*`
  - Register validation exports in `src/gateway/protocol/index.ts`
  - Wire handler in `src/gateway/server-methods/*.ts`

## 5) Testing Rules

For non-trivial changes, include tests at the same layer as the change:

- CLI behavior: `src/cli/**/*.test.ts`
- Gateway methods/protocol: `src/gateway/**/*.test.ts` or `*.e2e.test.ts`
- Agent/session logic: `src/commands/**/*.test.ts`, `src/agents/**/*.test.ts`
- Channel behavior: channel-local tests in `src/<channel>/` or `extensions/<channel>/src`
- Plugin behavior: `src/plugins/**/*.test.ts` and extension-local tests

Baseline verification commands:

- `pnpm build`
- `pnpm check`
- `pnpm test`

Use targeted test files first, then expand.

## 6) Common Drift Risks

- Doc drift vs source: verify file paths and symbol names before referencing.
- Alias confusion in model/provider ids: normalize via `normalizeProviderId` in `src/agents/model-selection.ts`.
- Multi-agent routing bugs: validate bindings and account normalization (`src/routing/resolve-route.ts`).
- Plugin command/method collisions: check registry guards in `src/plugins/registry.ts`.

## 7) Done Criteria for AI Agents

A change is done only when:

1. Code is wired through canonical seams (not shortcuts).
2. Config/protocol/schema updates are complete and validated.
3. Relevant tests are added or updated.
4. At least one execution path is validated locally (tests or command runs).
5. No unrelated files are modified.
