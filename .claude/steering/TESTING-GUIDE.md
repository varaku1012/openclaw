# OpenClaw Testing Guide

## Test Framework

- **vitest** (v4.x) for all TypeScript tests
- **Parallel test runner**: `scripts/test-parallel.mjs` - Custom parallel execution
- **Coverage**: v8 provider, 70% threshold for lines/functions/statements; 55% for branches
- **Config**: `vitest.config.ts` (main), `vitest.e2e.config.ts` (e2e), `vitest.live.config.ts` (live)
- **Split configs** (parallel optimization): `vitest.unit.config.ts`, `vitest.gateway.config.ts`, `vitest.extensions.config.ts`
- **Pool**: forks (better isolation than threads)
- **Workers**: 2-3 on Windows CI, 4-16 on others (CPU-based)
- **Timeouts**: 120s default, 180s hookTimeout on Windows

## Test Categories

### Unit Tests (Fast, No Network)

```bash
pnpm test                    # Run all unit tests (parallel)
pnpm vitest run src/path/to/file.test.ts  # Single file
pnpm test:watch              # Watch mode
pnpm test:coverage           # With coverage
```

- Colocated with source: `foo.ts` -> `foo.test.ts`
- No network calls - mock all external dependencies
- Fast execution - should complete in seconds
- Pattern in `vitest` config: `src/**/*.test.ts`

### E2E Tests (Integration)

```bash
pnpm test:e2e                # Run E2E tests
```

- Config: `vitest.e2e.config.ts`
- Test file pattern: `*.e2e.test.ts`
- Tests gateway startup, client connections, protocol flows
- May use local WebSocket connections

### Live Tests (Real API Keys)

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live
```

- Config: `vitest.live.config.ts`
- Test file pattern: `*.live.test.ts`
- Requires real API keys (Anthropic, OpenAI, etc.)
- Tests actual model interactions, provider auth flows
- Never run in CI without explicit opt-in

### Docker Tests

```bash
pnpm test:docker:all         # All Docker tests
pnpm test:docker:live-models # Live model tests in Docker
pnpm test:docker:onboard     # Onboarding flow test
pnpm test:docker:plugins     # Plugin loading tests
```

- Isolated environment testing
- Scripts in `scripts/e2e/*.sh`

## Test Patterns

### Standard Unit Test

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";

describe("moduleName", () => {
  let deps: ReturnType<typeof createDefaultDeps>;

  beforeEach(() => {
    deps = createDefaultDeps();
    deps.logger = { info: vi.fn(), warn: vi.fn(), error: vi.fn() };
  });

  it("does the expected thing", () => {
    const result = myFunction("input", deps);
    expect(result).toBe("expected");
  });

  it("handles errors", () => {
    deps.readFile = vi.fn().mockRejectedValue(new Error("fail"));
    expect(() => myFunction("bad", deps)).toThrow("fail");
  });
});
```

### Testing with Dependency Injection

```typescript
describe("processMessage", () => {
  it("routes to correct handler", async () => {
    const mockGateway = {
      handleInbound: vi.fn().mockResolvedValue(undefined),
    };
    const deps = { ...createDefaultDeps(), gateway: mockGateway };

    await processMessage({ text: "hello", from: "user1" }, deps);

    expect(mockGateway.handleInbound).toHaveBeenCalledWith(
      expect.objectContaining({ text: "hello", from: "user1" }),
    );
  });
});
```

### Testing Agent Tools

```typescript
describe("restaurant_menu tool", () => {
  const mockMenu = [
    { id: "1", name: "Pasta", price: 12.99, category: "entree" },
    { id: "2", name: "Salad", price: 8.99, category: "appetizer" },
  ];

  it("searches by category", async () => {
    const result = await menuTool.execute(
      { action: "search", category: "entree" },
      { config: testConfig },
    );
    expect(result.items).toHaveLength(1);
    expect(result.items[0].name).toBe("Pasta");
  });

  it("returns empty for no matches", async () => {
    const result = await menuTool.execute(
      { action: "search", query: "nonexistent" },
      { config: testConfig },
    );
    expect(result.items).toHaveLength(0);
  });
});
```

### Testing Plugin Registration

```typescript
describe("my-plugin", () => {
  it("registers expected tools", () => {
    const registered: string[] = [];
    const mockApi = {
      registerTool: vi.fn((tool) => registered.push(tool.name)),
      registerCommand: vi.fn(),
      registerHook: vi.fn(),
      // ... other API methods
    } as unknown as OpenClawPluginApi;

    registerPlugin(mockApi);

    expect(registered).toContain("my_tool");
    expect(mockApi.registerCommand).toHaveBeenCalled();
  });
});
```

### Testing Gateway Protocol

```typescript
describe("gateway protocol", () => {
  it("validates send params", () => {
    const valid = validateSendParams({
      target: "whatsapp:123@s.whatsapp.net",
      text: "Hello",
    });
    expect(valid).toBe(true);
  });

  it("rejects invalid params", () => {
    const valid = validateSendParams({
      target: "",  // Invalid: empty target
    });
    expect(valid).toBe(false);
  });
});
```

## Mocking Patterns

### Mock External APIs

```typescript
const mockFetch = vi.fn().mockResolvedValue({
  ok: true,
  json: () => Promise.resolve({ data: "result" }),
});

// Inject via deps or vi.mock
vi.mock("undici", () => ({
  fetch: mockFetch,
}));
```

### Mock File System

```typescript
import { vi } from "vitest";
import { vol } from "memfs";

vi.mock("node:fs/promises", () => vol.promises);

beforeEach(() => {
  vol.reset();
  vol.fromJSON({
    "/config/openclaw.json": JSON.stringify({ model: "claude-3.5-sonnet" }),
  });
});
```

### Mock Configuration

```typescript
const testConfig: OpenClawConfig = {
  model: "claude-3.5-sonnet",
  channels: {
    whatsapp: {
      enabled: true,
      dm: { allowFrom: ["1234567890"] },
    },
  },
};
```

## Parallel Test Runner

The custom runner (`scripts/test-parallel.mjs`) manages execution:

```
parallelRuns: [unit, extensions]  (run simultaneously)
serialRuns:   [gateway]           (always serial - avoids port conflicts)
```

Windows CI: all runs serial with 2-shard split to avoid memory/worker crashes.

Do NOT set test workers above 16.

## Test Isolation Utilities

### Temp Home (CRITICAL for file system tests)

```typescript
import { withTempHome } from "../../test/helpers/temp-home.js";

it("reads config", async () => {
  await withTempHome(async (home) => {
    // Runs in isolated HOME directory - never touches ~/.openclaw/
    // Auto-cleanup on completion
  });
});
```

### Deterministic Port Allocation (CRITICAL for multi-service tests)

```typescript
import { getDeterministicFreePortBlock } from "../test-utils/ports.js";

it("starts gateway and companion", async () => {
  const port = await getDeterministicFreePortBlock({ offsets: [0, 1, 2, 3] });
  // Per-worker sharding avoids EADDRINUSE in parallel tests
});
```

### Polling (prefer over sleep)

```typescript
import { pollUntil } from "../../test/helpers/poll.js";

await pollUntil(() => checkCondition(), { timeoutMs: 2000 });
```

### Channel Plugin Stubs

```typescript
import { createTestRegistry } from "../test-utils/channel-plugins.js";

const registry = createTestRegistry([stubPlugin("telegram")]);
```

## Advanced Mock Patterns

### vi.hoisted() for Shared Mock State

```typescript
// Use when mocks need to share state or interact with test code
const registryState = vi.hoisted(() => ({
  registry: { plugins: [], tools: [], channels: [] }
}));

vi.mock("./server-plugins.js", async () => {
  const { setActivePluginRegistry } = await import("../plugins/runtime.js");
  return {
    loadGatewayPlugins: (params) => {
      setActivePluginRegistry(registryState.registry);
      return { pluginRegistry: registryState.registry };
    }
  };
});
```

### Live Test Skip Pattern

```typescript
const LIVE = isTruthyEnvValue(process.env.OPENCLAW_LIVE_TEST);
const describeLive = LIVE ? describe : describe.skip;

describeLive("live provider test", () => {
  it("calls real API", async () => {
    // Only runs with OPENCLAW_LIVE_TEST=1
  }, 5 * 60 * 1000); // 5 minute timeout for live tests
});
```

## Coverage

```bash
pnpm test:coverage
```

Thresholds (in `vitest.config.ts`):
- Lines: 70%
- Functions: 70%
- Branches: 55%
- Statements: 70%

Coverage includes `src/**/*.ts`, excludes `src/**/*.test.ts`.

## Test File Naming

| Type | Pattern | Example |
|------|---------|---------|
| Unit | `*.test.ts` | `session-manager.test.ts` |
| E2E | `*.e2e.test.ts` | `gateway.e2e.test.ts` |
| Live | `*.live.test.ts` | `gateway-models.live.test.ts` |
| Docker | Shell scripts in `scripts/e2e/` | `onboard-docker.sh` |

## When to Write Tests

- All new tools need unit tests
- All new commands need unit tests
- Config schema changes need validation tests
- Protocol changes need schema validation tests
- Channel adapters need unit tests for each adapter method
- Pure test additions don't need changelog entries unless they alter user-facing behavior

## DO / DON'T Checklist

**DO:**
- Use `withTempHome()` for file system tests
- Use `pollUntil()` for async assertions (not `sleep`)
- Reset mocks in `beforeEach()`, cleanup in `afterEach()`/`afterAll()`
- Use `getDeterministicFreePortBlock()` for multi-service tests
- Test edge cases: timeouts, errors, race conditions
- Colocate tests with source (`foo.ts` -> `foo.test.ts`)
- Use `vi.hoisted()` for shared mock state
- Add explicit timeouts for slow tests (`{ timeout: 60_000 }`)

**DON'T:**
- Use arbitrary `sleep()` (causes flakiness)
- Touch real `~/.openclaw/` (use temp homes)
- Use OS free ports directly (use `getDeterministicFreePortBlock`)
- Leave resources uncleaned (add `afterEach`/`afterAll`)
- Skip auth tests (critical for production)
- Test only happy paths (test errors, timeouts, edge cases)

## Key Test Files Reference

| Purpose | File |
|---------|------|
| Global setup | `test/setup.ts` |
| Temp home helper | `test/helpers/temp-home.ts` |
| Polling helper | `test/helpers/poll.ts` |
| Port allocation | `src/test-utils/ports.ts` |
| Plugin stubs | `src/test-utils/channel-plugins.ts` |
| Gateway test helpers | `src/gateway/test-helpers.server.ts` |
