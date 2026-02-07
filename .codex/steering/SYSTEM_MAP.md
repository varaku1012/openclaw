# System Map

Last validated: 2026-02-06

## 1) Runtime Entry Chain

- Packaged CLI binary entry: `openclaw.mjs`
- Runtime bootstrap (dev/source): `src/entry.ts`
- CLI runner: `src/cli/run-main.ts`
- Program assembly: `src/cli/program/build-program.ts`
- Command registry and lazy subcommand loading: `src/cli/program/command-registry.ts`, `src/cli/program/register.subclis.ts`

Notes:

- Subcommands are lazily registered by default.
- Plugin CLI commands are injected via `src/plugins/cli.ts`.

## 2) Control Plane (Gateway)

Primary server composition is in `src/gateway/server.impl.ts`:

- Loads and validates config snapshot
- Applies plugin auto-enable rules
- Initializes plugin registry and channel plugins
- Resolves runtime network/auth/tailscale config
- Starts WS + HTTP surfaces
- Initializes cron, node registry, channel manager, hooks, health/presence events

Key files:

- Public API export: `src/gateway/server.ts`
- Core method dispatch and authorization: `src/gateway/server-methods.ts`
- Available methods/events list: `src/gateway/server-methods-list.ts`
- WS connection lifecycle: `src/gateway/server/ws-connection.ts`
- WS message handshake/auth/dispatch: `src/gateway/server/ws-connection/message-handler.ts`
- HTTP surface router: `src/gateway/server-http.ts`
- Protocol schemas and validators: `src/gateway/protocol/schema.ts`, `src/gateway/protocol/index.ts`

Core methods include:

- `connect`, `health`, `status`, `channels.*`, `send`, `agent`, `chat.*`
- `config.*`, `wizard.*`, `cron.*`, `sessions.*`, `models.list`
- `node.*`, `device.*`, `exec.approvals.*`, `skills.*`, `update.run`

## 3) Agent Runtime Layer

Main request path for agent execution:

- CLI/Gateway enters `src/commands/agent.ts`
- Session resolution: `src/commands/agent/session.ts`
- Inbound preprocessing and directives: `src/auto-reply/reply/get-reply.ts`
- Embedded runner: `src/agents/pi-embedded-runner/*`
- Built-in tool assembly: `src/agents/openclaw-tools.ts`

CLI backend path (when provider is CLI-backed):

- Backend config/normalization: `src/agents/cli-backends.ts`
- CLI agent execution wrapper: `src/agents/cli-runner.ts`

Model selection and allowlist behavior:

- `src/agents/model-selection.ts`
- `src/agents/model-catalog.ts`
- `src/agents/model-fallback.ts`

## 4) Channel Layer

Channel contracts are plugin-driven:

- Plugin contract: `src/channels/plugins/types.plugin.ts`
- Adapter contracts: `src/channels/plugins/types.adapters.ts`
- Runtime registry access: `src/channels/plugins/index.ts`
- Lightweight shared behavior map: `src/channels/dock.ts`
- Channel id/meta normalization: `src/channels/registry.ts`

Outbound delivery is unified through channel outbound adapters:

- `src/infra/outbound/deliver.ts`

WhatsApp/web path specifics:

- Re-export barrel: `src/channel-web.ts`
- Inbound monitor/reconnect/heartbeat: `src/web/auto-reply/monitor.ts`, `src/web/auto-reply/heartbeat-runner.ts`
- Outbound send: `src/web/outbound.ts`

## 5) Plugin and Extension System

Discovery and load:

- Candidate discovery: `src/plugins/discovery.ts`
- Manifest registry: `src/plugins/manifest-registry.ts`
- Loader and registration: `src/plugins/loader.ts`
- Registry internals: `src/plugins/registry.ts`

Runtime API available to plugins:

- `src/plugins/runtime/index.ts`
- Public SDK exports: `src/plugin-sdk/index.ts`

Plugin metadata contract:

- `extensions/<extension-id>/openclaw.plugin.json` in each extension root
- Mandatory `configSchema` (enforced during validation)

Extension workspace packages live in `extensions/*` and are included by `pnpm-workspace.yaml`.

## 6) Skills and Memory

Skills:

- Skill loading and prompt composition: `src/agents/skills/workspace.ts`
- Skill env overrides and gating: `src/agents/skills/env-overrides.ts`, `src/agents/skills/config.ts`
- Top-level exports: `src/agents/skills.ts`
- Bundled skills: `skills/*/SKILL.md`

Memory:

- Memory manager/index/search: `src/memory/manager.ts`
- Tool wrappers: `src/agents/tools/memory-tool.ts`
- Memory manager resolver: `src/memory/search-manager.ts`

## 7) Config and State

Config system:

- Root exports: `src/config/config.ts`
- Main schema: `src/config/zod-schema.ts`
- Type composition: `src/config/types.ts`
- Path resolution and state dir logic: `src/config/paths.ts`

Important defaults:

- State dir: `~/.openclaw` (or `OPENCLAW_STATE_DIR`)
- Config: `~/.openclaw/openclaw.json` (JSON5)
- Agent sessions: `~/.openclaw/agents/<agentId>/sessions`
- Node host config: `~/.openclaw/node.json`

## 8) Routing and Multi-Agent Isolation

Routing core:

- Binding parsing/helpers: `src/routing/bindings.ts`
- Route resolution: `src/routing/resolve-route.ts`
- Session key construction: `src/routing/session-key.ts`

Multi-agent isolation depends on:

- `agents.list[]` agent ids/workspaces
- `bindings[]` mapping `(channel, accountId, peer/guild/team)` to agent id
- Per-agent `agentDir`, workspace, session storage

## 9) Automation Surfaces

- Cron service and handlers: `src/cron/`, `src/gateway/server-methods/cron.ts`
- Webhooks and mapping: `src/gateway/hooks.ts`, `src/gateway/server-http.ts`
- Internal hooks/plugin lifecycle: `src/hooks/`, `src/plugins/types.ts` (plugin hooks)

## 10) UI and Platform Apps

Control UI:

- Frontend source: `ui/src/`
- Build/test scripts: `ui/package.json`
- Served through gateway HTTP path via `src/gateway/control-ui.ts` and `src/gateway/server-http.ts`

Native apps:

- iOS: `apps/ios/`
- macOS: `apps/macos/`
- Android: `apps/android/`
- Shared Apple framework: `apps/shared/OpenClawKit/`

## 11) Quality Gates and Test Topology

Primary scripts:

- `pnpm build`
- `pnpm check`
- `pnpm test`
- `pnpm test:e2e`
- `pnpm test:coverage`

Config files:

- Unit/integration: `vitest.config.ts`
- E2E: `vitest.e2e.config.ts`
- Live: `vitest.live.config.ts`

Coverage thresholds in unit config: lines/functions/statements 70%, branches 55%.
