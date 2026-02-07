# AI Agent Workflow

Last validated: 2026-02-07

Use this workflow for reliable changes in this repo.

## 1) Preflight (Always)

1. Read `AGENTS.md`.
2. Read this steering pack in order from `README.md`.
3. Identify affected layer(s): CLI, gateway, agent runtime, channel/plugin, skills, UI.
4. Locate canonical files before editing.

## 2) Change Planning Template

For each task, answer:

- User-facing behavior change:
- Primary module:
- Secondary modules:
- Config/protocol impact:
- Security impact:
- Test scope:

If config/protocol changes, plan schema + handler + tests together.

## 3) Editing Protocol

- Prefer small, surgical edits.
- Preserve existing naming and module boundaries.
- Keep changes inside one layer unless cross-layer work is required.
- If touching shared contracts, update all call sites in same commit.

## 4) Validation Protocol

Minimum:

- `pnpm build`
- Relevant targeted tests

Standard for most PRs:

- `pnpm check`
- `pnpm test`
- Optionally use split test configs for faster feedback:
  - `vitest.unit.config.ts`
  - `vitest.gateway.config.ts`
  - `vitest.extensions.config.ts`

If UI changed:

- `pnpm ui:build`
- `pnpm test:ui`

If gateway protocol/method changed:

- Add/update gateway tests and at least one integration/e2e assertion.
- Run `pnpm protocol:gen`.
- Run `pnpm protocol:gen:swift` when companion protocol models are affected.

For filesystem/network-sensitive tests:

- Use `withTempHome` from `test/helpers/temp-home.ts`.
- Prefer `pollUntil` from `test/helpers/poll.ts` over arbitrary sleeps.
- Use `getDeterministicFreePortBlock` from `src/test-utils/ports.ts` for multi-service tests.

## 5) Tool Authoring Workflow

When adding or modifying tools:

1. Keep input schema root as an object; avoid top-level unions.
2. Prefer `stringEnum` / `optionalStringEnum` schema helpers.
3. Use parsing/result helpers from `src/agents/tools/common.ts`.
4. For side-effecting operations, split into preview and commit actions.

## 6) Vertical Feature Workflow

For restaurants/marketing/dealership features:

1. Start in extension plugin package.
2. Build tool contracts first (read-only then commit actions).
3. Add skill instructions for correct usage sequence.
4. Bind specific tenant channels/accounts via `bindings[]`.
5. Add automation only after manual path is stable.

## 7) Safety Workflow for Side Effects

Any external side effect (order, campaign publish, finance quote commit, etc.) should use:

1. `preview` action/tool
2. explicit user confirmation
3. `commit` action/tool
4. audit log event/result

Do not compress these into one opaque tool call.

## 8) PR Completion Checklist

- Behavior implemented and documented in code comments only where needed.
- Tests added/updated and executed.
- No unrelated files changed.
- No generated outputs manually edited.
- File references in docs are valid.
