# OpenClaw Steering Context for Coding Agents

Last validated: 2026-02-06

This folder is a verified steering pack for AI coding agents working in this repo.
Use it to avoid architectural drift, pick the right extension seams, and ship vertical features safely.

## Read Order

1. `SYSTEM_MAP.md`
2. `ENGINEERING_GUARDRAILS.md`
3. `EXTENSION_SURFACES.md`
4. `VERTICALIZATION_BLUEPRINT.md`
5. `AI_AGENT_WORKFLOW.md`
6. `VALIDATION.md`

## Scope

This steering context is grounded in current source and docs, especially:

- Runtime and CLI entry: `openclaw.mjs`, `src/entry.ts`, `src/cli/run-main.ts`
- Gateway control plane: `src/gateway/server.impl.ts`, `src/gateway/server-methods.ts`, `src/gateway/protocol/`
- Agent runtime: `src/commands/agent.ts`, `src/auto-reply/reply/get-reply.ts`, `src/agents/`
- Channels and outbound: `src/channels/`, `src/infra/outbound/deliver.ts`, `extensions/*`
- Skills/plugins: `src/agents/skills.ts`, `src/plugins/loader.ts`, `src/plugin-sdk/index.ts`
- Config/schema: `src/config/`
- UI/apps: `ui/`, `apps/`

## Known Drift Corrected Here

- `AGENTS.md` references `src/provider-web.ts`, but this file does not exist in current repo.
  WhatsApp/web channel runtime is under `src/web/` and re-exported via `src/channel-web.ts`.

