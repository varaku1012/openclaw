# OpenClaw Coding Conventions

## Language & Runtime

- **TypeScript** with strict typing. ESM modules only (`"type": "module"` in package.json).
- **Node 22+** required. Target ES2024+.
- **Avoid `any`** - Use proper generics, `unknown`, or specific types.
- **Import extensions**: Always use `.js` extensions in imports (ESM requirement).

## File Organization

- **Max ~500 LOC per file** - Enforced by `pnpm check:loc`. Split large files.
- **Colocated tests**: `foo.ts` -> `foo.test.ts` in the same directory.
- **Barrel exports**: Use `index.ts` for module public API.
- **Type-only imports**: Use `import type { ... }` when importing only types.

## Naming Conventions

| Entity | Convention | Example |
|--------|-----------|---------|
| Files | kebab-case | `session-manager.ts` |
| Classes | PascalCase | `SessionManager` |
| Functions | camelCase | `resolveSessionKey` |
| Constants | UPPER_SNAKE_CASE | `DEFAULT_ACCOUNT_ID` |
| Types/Interfaces | PascalCase | `ChannelPlugin`, `OpenClawConfig` |
| Enums | PascalCase members | `ErrorCodes.InvalidParams` |
| Package name | lowercase | `openclaw` |
| CLI commands | kebab-case | `gateway-status` |
| Config keys | camelCase | `groupPolicy`, `allowFrom` |
| Tool names | snake_case | `restaurant_order` |
| Channel IDs | lowercase | `telegram`, `whatsapp` |
| Plugin IDs | kebab-case | `openclaw-restaurant` |

## Dependency Injection Pattern

The codebase uses a `createDefaultDeps` pattern for testability:

```typescript
// In implementation
export function createDefaultDeps() {
  return {
    readFile: fs.readFile,
    writeFile: fs.writeFile,
    logger: createLogger(),
  };
}

export type Deps = ReturnType<typeof createDefaultDeps>;

export async function processMessage(message: string, deps: Deps = createDefaultDeps()) {
  deps.logger.info(`Processing: ${message}`);
  // ...
}

// In tests
it("processes message", async () => {
  const deps = createDefaultDeps();
  deps.logger = { info: vi.fn(), warn: vi.fn(), error: vi.fn() };
  await processMessage("hello", deps);
  expect(deps.logger.info).toHaveBeenCalledWith("Processing: hello");
});
```

## Tool Schema Patterns

CRITICAL rules for agent tool input schemas:

```typescript
import { Type } from "@sinclair/typebox";
import { stringEnum, optionalStringEnum } from "openclaw/plugin-sdk";

// CORRECT - Use stringEnum for required enums
const schema = Type.Object({
  action: stringEnum(["create", "read", "update", "delete"]),
  priority: optionalStringEnum(["low", "medium", "high"]),
  name: Type.String(),
  count: Type.Optional(Type.Number()),
});

// WRONG - Never use Type.Union for string enums in tool schemas
const badSchema = Type.Object({
  action: Type.Union([Type.Literal("create"), Type.Literal("delete")]), // DON'T DO THIS
});
```

## Error Handling

- Use typed error classes for domain errors.
- Log errors with structured metadata via `tslog`.
- Never swallow errors silently - at minimum log them.
- Return user-friendly error messages from tool executions.

```typescript
try {
  await operation();
} catch (error) {
  logger.error("Operation failed", { error, context: { sessionKey } });
  throw error; // Re-throw unless you handle it
}
```

## Formatting & Linting

- **Linter**: oxlint (type-aware with `tsconfig.oxlint.json`)
- **Formatter**: oxfmt
- **Run before commits**: `pnpm check` (runs `tsgo && lint && format`)
- **Auto-fix**: `pnpm lint:fix`

## Terminal Output

- Use palette from `src/terminal/palette.ts` - **no hardcoded ANSI colors**.
- Use progress indicators from `src/cli/progress.ts` (osc-progress + @clack/prompts spinner).
- Use `chalk` for coloring (already a dependency), but prefer palette abstraction.

## Configuration

- Config files use **JSON5** format (comments allowed).
- Config validation uses **Zod** schemas.
- Config types are in `src/config/types*.ts`.
- New config sections need: Zod schema, TypeScript type, and validation integration.

## Commits & Branching

- Use `scripts/committer "<msg>" <file...>` for scoped commits.
- Messages: concise, action-oriented (e.g., `CLI: add verbose flag to send`).
- Never commit secrets, API keys, phone numbers.

## Patched Dependencies

Any dependency in `pnpm.patchedDependencies` MUST use exact version (no `^` or `~`). Check `package.json` patches before upgrading.

## SwiftUI (macOS/iOS Apps)

- Prefer `@Observable`/`@Bindable` over `ObservableObject`/`@StateObject`.
- Share code via `OpenClawKit` framework in `apps/shared/`.

## Module Boundaries

- **Plugins/extensions** should only import from `openclaw/plugin-sdk` - never from internal paths.
- **Channel implementations** use the channel adapter interfaces from `src/channels/plugins/types.ts`.
- **Shared code** should import from `src/channels/dock.ts` and `src/channels/registry.ts`, not from the plugin registry directly.

## Engineering Guardrails (Non-Negotiables)

These rules must NEVER be violated by AI coding agents:

1. **Never edit `dist/` directly** — always modify source and rebuild.
2. **Never bypass schema validation** — do not add ad-hoc config reads; extend `src/config/types*.ts` + Zod schema files.
3. **Never hardcode new commands** — wire through CLI registrars (`src/cli/program/command-registry.ts`, `src/cli/program/register.subclis.ts`).
4. **Never import heavy channel modules into shared paths** — use `src/channels/dock.ts` and `src/channels/registry.ts` abstractions.
5. **Keep `src/plugin-sdk/index.ts` stable and additive** — it is the public API surface; breaking changes break all extensions.
6. **Route all outbound sends through adapters** — use `src/infra/outbound/deliver.ts`, not channel-specific sends.
7. **Never bypass gateway auth** — WS handshake must pass auth/role checks via `src/gateway/server/ws-connection/message-handler.ts`.
8. **Never bypass node command policy** — allowlist checks in `src/gateway/node-command-policy.ts` are mandatory.
9. **Hook/webhook auth checks are mandatory** — never remove token verification in `src/gateway/server-http.ts`.

## AI Agent Workflow

### Change Planning Template

Before starting any non-trivial change, answer:
- **User-facing behavior change**: What changes for users?
- **Primary module**: Which `src/` module owns this?
- **Secondary modules**: What else is affected?
- **Config/protocol impact**: Any schema changes needed?
- **Security impact**: Auth, allowlists, sandboxing affected?
- **Test scope**: Unit, E2E, or live tests needed?

If config/protocol changes, plan schema + handler + tests together in the same commit.

### Editing Protocol

- Prefer small, surgical edits.
- Preserve existing naming and module boundaries.
- Keep changes inside one layer unless cross-layer work is required.
- If touching shared contracts, update all call sites in the same commit.

### Safety Workflow for Side Effects

Any tool that triggers an external side effect (order placement, campaign publish, finance quote commit) MUST use:

1. **Preview** action/tool — show what will happen
2. **Explicit user confirmation** — require the user to approve
3. **Commit** action/tool — execute the action
4. **Audit log** — record the result

Never compress these into one opaque tool call.

### Done Criteria

A change is complete only when:
1. Code is wired through canonical seams (not shortcuts)
2. Config/protocol/schema updates are complete and validated
3. Relevant tests are added or updated
4. At least one execution path is validated (`pnpm build && pnpm check && pnpm test`)
5. No unrelated files are modified

## Where to Add New Functionality

| What | Where | Key Files |
|------|-------|-----------|
| CLI command | `src/commands/` | `src/cli/program/command-registry.ts`, `register.subclis.ts` |
| Gateway RPC method | `src/gateway/server-methods/` | `protocol/schema/*.ts`, `server-methods-list.ts` |
| Channel | `extensions/<id>/` | `src/channels/plugins/types.plugin.ts`, `src/plugin-sdk/index.ts` |
| Agent tool (core) | `src/agents/tools/` | `src/agents/openclaw-tools.ts` |
| Agent tool (plugin) | `extensions/<id>/src/tools/` | `api.registerTool(...)` |
| Model provider | `src/providers/` or plugin | `src/agents/model-selection.ts`, `model-catalog.ts` |
| Skill | `skills/` or plugin | `src/agents/skills/workspace.ts`, `SKILL.md` |
| Webhook/cron | `src/cron/` or `src/gateway/hooks.ts` | `server-http.ts`, `hooks-mapping.ts` |
| UI change | `ui/src/` | `src/gateway/control-ui.ts` |
| Multi-agent routing | `src/routing/` | `bindings.ts`, `resolve-route.ts`, `session-key.ts` |
