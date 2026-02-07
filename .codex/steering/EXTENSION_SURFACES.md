# Extension Surfaces

Last validated: 2026-02-06

Use this map to choose the correct place for new functionality.

## 1) Add a New CLI Capability

Primary touchpoints:

- Command registration: `src/cli/program/command-registry.ts`
- Lazy subcommand loading: `src/cli/program/register.subclis.ts`
- Command implementation: `src/commands/` or `src/cli/<domain>-cli.ts`

Validation:

- Add/adjust command tests under `src/cli/` or `src/commands/`.
- Smoke run with `pnpm openclaw <command> --help`.

## 2) Add a New Gateway RPC Method

Primary touchpoints:

- Schema/type: `src/gateway/protocol/schema/*.ts`
- Validator exports: `src/gateway/protocol/index.ts`
- Method handler: `src/gateway/server-methods/<domain>.ts`
- Method registration list: `src/gateway/server-methods-list.ts` (if method should be discoverable)
- Authorization scopes: `src/gateway/server-methods.ts` (READ/WRITE/ADMIN decisions)

Validation:

- Add unit tests under `src/gateway/server-methods/*.test.ts`.
- Add e2e where behavior crosses WS/session boundaries.

## 3) Add a New Channel

Recommended path: extension package under `extensions/<channel-id>/`.

Required components:

- `extensions/<channel-id>/package.json` with `openclaw.extensions` entry
- `extensions/<channel-id>/openclaw.plugin.json` with `id`, `channels`, `configSchema`
- Entry module `extensions/<channel-id>/index.ts` registering `ChannelPlugin` through plugin API

Contracts and helpers:

- Channel plugin contract: `src/channels/plugins/types.plugin.ts`
- Adapter contracts: `src/channels/plugins/types.adapters.ts`
- Public SDK surface: `src/plugin-sdk/index.ts`

Optional core updates:

- Add lightweight metadata/alias order in `src/channels/registry.ts` (if becoming a core-first channel experience)
- Add dock behavior in `src/channels/dock.ts` for shared logic defaults

## 4) Add a New Agent Tool

Two options:

- Core tool: add in `src/agents/tools/` and include in `src/agents/openclaw-tools.ts`
- Plugin tool: register via `api.registerTool(...)` in extension

Important:

- Use schema patterns compatible with current tool framework.
- Respect tool policy gating (`tools.allow`, `tools.deny`, per-agent overrides).

## 5) Add a New Model Provider

Two options:

- Core provider wiring in `src/providers/` + model config paths
- Plugin provider via `api.registerProvider(...)`

Key files to align:

- Provider normalization and selection: `src/agents/model-selection.ts`
- Model catalog/compat rules: `src/agents/model-catalog.ts`, `src/agents/model-compat.ts`

## 6) Add Workflow Automation

### Webhook-driven

- HTTP entry path: `src/gateway/server-http.ts`
- Hook normalization and mapping: `src/gateway/hooks.ts`, `src/gateway/hooks-mapping.ts`

### Scheduled

- Cron domain: `src/cron/`
- Gateway methods: `src/gateway/server-methods/cron.ts`

## 7) Add or Evolve Skills

Primary touchpoints:

- Core skill logic: `src/agents/skills/workspace.ts`
- Skill config and gating: `src/agents/skills/config.ts`
- Skill env injection: `src/agents/skills/env-overrides.ts`
- Bundled skills: `skills/*/SKILL.md`

Plugin-shipped skills:

- Declare in extension manifest `extensions/<extension-id>/openclaw.plugin.json` via `skills` array.

## 8) Add Vertical Business Integrations (CRM/POS/DMS/etc.)

Preferred sequence:

1. Add integration client in extension package `extensions/<vertical>/src/integrations/`.
2. Expose controlled operations as plugin tools.
3. Add slash-like plugin commands for deterministic workflows.
4. Add webhook/cron connectors if external events or timed workflows are needed.
5. Add skills to teach usage patterns and safety confirmation steps.

## 9) UI Changes

Primary touchpoints:

- UI source: `ui/src/`
- Gateway serving path: `src/gateway/control-ui.ts`, `src/gateway/server-http.ts`

Validation:

- `pnpm ui:build`
- `pnpm test:ui`

## 10) Multi-Agent Routing and Tenant Isolation

When adding tenant/vertical separation behavior, align with:

- Agent identity/workspace resolution: `src/agents/agent-scope.ts`
- Binding routing: `src/routing/resolve-route.ts`
- Binding helpers: `src/routing/bindings.ts`
- Session key semantics: `src/routing/session-key.ts`

Do not introduce alternate routing stores outside these modules.
