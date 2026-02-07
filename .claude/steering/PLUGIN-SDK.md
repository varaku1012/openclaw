# OpenClaw Plugin SDK Guide

## Overview

The Plugin SDK (`src/plugin-sdk/index.ts`) is the public API surface for building extensions. All exports from this module are considered stable and available to extension developers.

## Plugin Entry Point

Every plugin is an extension package in `extensions/<name>/` with this structure:

```
extensions/<name>/
  package.json              # Must include openclaw.extensions array
  openclaw.plugin.json      # Plugin metadata (id, name, description)
  index.ts                  # Entry point
  src/                      # Implementation files
```

### package.json

```json
{
  "name": "@openclaw/<name>",
  "version": "2026.2.1",
  "type": "module",
  "devDependencies": {
    "openclaw": "workspace:*"
  },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

IMPORTANT: Use `devDependencies` or `peerDependencies` for `openclaw` dependency, NOT `dependencies`. Using `workspace:*` in `dependencies` breaks npm install.

### openclaw.plugin.json

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Description of what the plugin does"
}
```

### index.ts (Simple form)

```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

export default function register(api: OpenClawPluginApi) {
  // Register tools, hooks, commands, etc.
}
```

### index.ts (Full definition form)

```typescript
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk";

const plugin: OpenClawPluginDefinition = {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  configSchema: { /* Zod or custom schema */ },
  register(api) {
    // Registration phase - register capabilities
  },
  activate(api) {
    // Activation phase - start services
  }
};
export default plugin;
```

## Registration API (OpenClawPluginApi)

### registerTool(tool, opts?)

Register an agent tool that the AI can invoke:

```typescript
import { stringEnum, optionalStringEnum } from "openclaw/plugin-sdk";
import { Type } from "@sinclair/typebox";

const myTool = {
  name: "my_tool",
  description: "Does something useful",
  inputSchema: Type.Object({
    action: stringEnum(["create", "update", "delete"]),  // NOT Type.Union!
    target: Type.String(),
    priority: optionalStringEnum(["low", "medium", "high"]),
  }),
  async execute(input, context) {
    // Tool implementation
    return { result: "success" };
  }
};

api.registerTool(myTool);
```

CRITICAL: Never use `Type.Union` for string enums in tool schemas. Always use `stringEnum()` or `optionalStringEnum()` from the SDK.

### registerCommand(command)

Register a chat command that bypasses the LLM agent:

```typescript
api.registerCommand({
  name: "status",                    // Without leading /
  description: "Show system status",
  acceptsArgs: false,
  requireAuth: true,                 // Default: true
  handler: async (ctx) => {
    // ctx.senderId, ctx.channel, ctx.isAuthorizedSender, ctx.args, ctx.config
    return { text: "System is running" };
  }
});
```

### registerHook(events, handler, opts?)

Register lifecycle hooks:

```typescript
api.registerHook("message_received", async (event) => {
  console.log(`Message from ${event.from}: ${event.content}`);
});

// Multiple events
api.registerHook(["session_start", "session_end"], async (event) => {
  // Handle both events
});
```

### registerChannel(registration)

Register a new messaging channel:

```typescript
import type { ChannelPlugin } from "openclaw/plugin-sdk";

const myChannel: ChannelPlugin = {
  id: "mychannel",
  meta: { name: "My Channel", icon: "chat", color: "#007AFF" },
  capabilities: { chatTypes: ["direct", "group"] },
  config: { /* ChannelConfigAdapter */ },
  // ... other adapters
};

api.registerChannel({ plugin: myChannel });
```

### registerProvider(provider)

Register a new model provider:

```typescript
api.registerProvider({
  id: "my-llm",
  label: "My LLM Provider",
  envVars: ["MY_LLM_API_KEY"],
  auth: [{
    id: "api-key",
    label: "API Key",
    kind: "api_key",
    run: async (ctx) => ({
      profiles: [{ profileId: "default", credential: { type: "api_key", apiKey: "..." } }]
    })
  }]
});
```

### registerService(service)

Register a background service:

```typescript
api.registerService({
  id: "my-service",
  start: async (ctx) => {
    // ctx.config, ctx.workspaceDir, ctx.stateDir, ctx.logger
    // Start background processing
  },
  stop: async (ctx) => {
    // Cleanup
  }
});
```

### registerHttpRoute({ path, handler })

Register an HTTP endpoint on the gateway:

```typescript
import { normalizePluginHttpPath, registerPluginHttpRoute } from "openclaw/plugin-sdk";

api.registerHttpRoute({
  path: normalizePluginHttpPath(api.id, "/webhook"),
  handler: async (req, res) => {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ ok: true }));
  }
});
```

### registerGatewayMethod(method, handler)

Register a custom gateway RPC method:

```typescript
api.registerGatewayMethod("my_plugin.action", async (params, opts) => {
  return { success: true, data: params };
});
```

### registerCli(registrar)

Register CLI subcommands:

```typescript
api.registerCli(({ program, config, logger }) => {
  program
    .command("my-command")
    .description("My custom command")
    .action(async () => {
      logger.info("Running my command");
    });
});
```

### on(hookName, handler, opts?)

Register plugin lifecycle hooks:

```typescript
api.on("before_agent_start", async (event, ctx) => {
  return { prependContext: "Additional context for the agent" };
});

api.on("message_sending", async (event, ctx) => {
  // Modify or cancel outgoing messages
  return { content: event.content.toUpperCase() };
  // OR: return { cancel: true };
});

api.on("before_tool_call", async (event, ctx) => {
  // Block or modify tool calls
  if (event.toolName === "dangerous_tool") {
    return { block: true, blockReason: "This tool is disabled" };
  }
});
```

## Available Plugin Hooks

| Hook | When | Can Modify? |
|------|------|-------------|
| `before_agent_start` | Before LLM invocation | Yes (systemPrompt, prependContext) |
| `agent_end` | After LLM completes | No (observe only) |
| `before_compaction` | Before context summarization | No |
| `after_compaction` | After context summarization | No |
| `message_received` | Inbound message arrives | No |
| `message_sending` | Before sending reply | Yes (content, cancel) |
| `message_sent` | After reply sent | No |
| `before_tool_call` | Before tool execution | Yes (params, block) |
| `after_tool_call` | After tool execution | No |
| `tool_result_persist` | Before saving tool result | Yes (message) |
| `session_start` | New session begins | No |
| `session_end` | Session ends | No |
| `gateway_start` | Gateway boots | No |
| `gateway_stop` | Gateway shuts down | No |

## Key SDK Exports

### Schema Helpers
- `stringEnum(values)` - Required string enum for tool schemas
- `optionalStringEnum(values)` - Optional string enum for tool schemas
- `buildChannelConfigSchema()` - Build config schema for channels

### Channel Utilities
- `getChatChannelMeta(id)` - Get channel metadata
- `resolveChannelMediaMaxBytes()` - Get media size limits
- `createTypingCallbacks()` - Typing indicator helpers
- `createReplyPrefixContext()` - Reply prefix helpers

### Configuration
- `normalizeAccountId()` - Normalize account identifiers
- `DEFAULT_ACCOUNT_ID` - Default account ID constant
- `normalizeE164()` - Normalize phone numbers (E.164)

### Media
- `detectMime()` / `extensionForMime()` - MIME detection
- `loadWebMedia()` - Load media from URLs

### Logging
- `registerLogTransport()` - Register custom log destinations
- `emitDiagnosticEvent()` / `onDiagnosticEvent()` - Diagnostic events
