# OpenClaw Architecture - Deep Dive

## Module Dependency Graph

```
                    src/config/         (Zod schemas, types, IO, validation)
                        |
                        v
                    src/gateway/        (WebSocket server, protocol, RPC)
                    /    |    \
                   /     |     \
                  v      v      v
        src/agents/  src/channels/  src/sessions/
            |            |              |
            v            v              v
     src/providers/  src/auto-reply/  src/routing/
            |            |
            v            v
     src/media/    src/commands/
     src/media-understanding/
     src/browser/
     src/tts/
     src/memory/
```

## Gateway (src/gateway/)

The gateway is the **central nervous system**. It is a WebSocket server (default port 18789) that:

1. **Manages client connections** - Companion apps, CLI, and channel bridges connect as WebSocket clients
2. **Routes messages** - From channels to the agent and back
3. **Handles RPC** - JSON-RPC-like request/response pattern over WebSocket frames
4. **Manages sessions** - Tracks conversation state per channel/user pair
5. **Serves HTTP** - Webhook endpoints, control UI, OpenAI-compatible API

### Protocol Structure (`src/gateway/protocol/`)

The protocol uses TypeBox schemas (compiled via AJV) for validation:

- **Frames**: `RequestFrame`, `ResponseFrame`, `EventFrame` - Wire format
- **Methods**: `send`, `poll`, `agent`, `chat.send`, `config.get`, `sessions.list`, etc. (100+ methods)
- **Events**: `agent_event`, `chat_event`, `tick`, `shutdown` - Server-push events
- **Schema files**: `src/gateway/protocol/schema/*.ts` - Organized by domain

Key gateway files:
- `server-methods.ts` / `server-methods/*.ts` - RPC method handlers
- `server-chat.ts` - Chat/conversation management
- `server-lanes.ts` - Message queuing/lane system
- `server-channels.ts` - Channel lifecycle management
- `server-http.ts` - HTTP endpoint serving
- `client.ts` - Gateway client (for CLI/apps connecting to gateway)
- `boot.ts` - Gateway startup sequence

## Agent System (src/agents/)

Built on the `@mariozechner/pi-agent-core` and `@mariozechner/pi-coding-agent` libraries:

- **Agent runtime**: Orchestrates LLM conversations with tool calling
- **Tool definitions**: `src/agents/tools/` - Built-in tools (send messages, browse, search, etc.)
- **Tool schemas**: Uses TypeBox (`@sinclair/typebox`) - IMPORTANT: Avoid `Type.Union` in tool input schemas; use `stringEnum`/`optionalStringEnum` instead
- **Identity**: `src/agents/identity.ts` - Agent personality/system prompt management
- **Compaction**: `src/agents/compaction.ts` - Conversation summarization when context gets long
- **Auth profiles**: `src/agents/auth-profiles/` - Multi-provider auth management

## Channel System (src/channels/)

### Channel Plugin Interface

Every channel implements `ChannelPlugin<ResolvedAccount>` (defined in `src/channels/plugins/types.plugin.ts`):

```typescript
type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;                    // Unique channel identifier
  meta: ChannelMeta;                // Display name, icon, color
  capabilities: ChannelCapabilities; // What the channel supports
  config: ChannelConfigAdapter<RA>; // Config resolution
  setup?: ChannelSetupAdapter;      // First-time setup
  pairing?: ChannelPairingAdapter;  // QR/code pairing
  security?: ChannelSecurityAdapter;// DM/group access control
  groups?: ChannelGroupAdapter;     // Group chat handling
  outbound?: ChannelOutboundAdapter;// Sending messages out
  gateway?: ChannelGatewayAdapter;  // Gateway integration
  streaming?: ChannelStreamingAdapter; // Block streaming support
  threading?: ChannelThreadingAdapter; // Thread handling
  messaging?: ChannelMessagingAdapter; // Inbound message processing
  directory?: ChannelDirectoryAdapter; // Contact/group directory
  actions?: ChannelMessageActionAdapter; // Message actions (react, pin, etc.)
  heartbeat?: ChannelHeartbeatAdapter;   // Health check
  agentTools?: ChannelAgentToolFactory;  // Channel-specific tools
  // ... more adapters
};
```

### Channel Dock (src/channels/dock.ts)

Lightweight channel metadata for shared code paths. Each channel has a dock entry with:
- Capabilities (chat types, native commands, block streaming)
- Outbound config (text chunk limits)
- Streaming config (coalesce defaults)
- Group handling (mention requirements, tool policies)

### Channel Registry (src/channels/registry.ts)

Central registry that maps channel IDs to their plugins and docks. Channels are registered both statically (core channels) and dynamically (via plugin SDK).

## Plugin System (src/plugins/)

### Plugin API (OpenClawPluginApi)

The central registration API exposed to plugins:

```typescript
type OpenClawPluginApi = {
  registerTool(tool, opts?)         // Register agent tools
  registerHook(events, handler)     // Register lifecycle hooks
  registerHttpHandler(handler)      // Register HTTP handlers
  registerHttpRoute({path, handler})// Register HTTP routes
  registerChannel(registration)     // Register new channels
  registerGatewayMethod(method, handler) // Register gateway methods
  registerCli(registrar)           // Register CLI commands
  registerService(service)         // Register background services
  registerProvider(provider)       // Register model providers
  registerCommand(command)         // Register chat commands
  on(hookName, handler)            // Register plugin lifecycle hooks
};
```

### Plugin Lifecycle Hooks

```
before_agent_start -> agent_end
before_compaction -> after_compaction
message_received -> message_sending -> message_sent
before_tool_call -> after_tool_call -> tool_result_persist
session_start -> session_end
gateway_start -> gateway_stop
```

### Plugin Definition

```typescript
// extensions/<name>/index.ts
export default {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  register(api: OpenClawPluginApi) {
    api.registerTool(myTool);
    api.registerCommand({ name: "mycmd", description: "...", handler: myHandler });
  }
};
```

### Extension Package Structure

```
extensions/<name>/
  package.json          # { "openclaw": { "extensions": ["./index.ts"] } }
  openclaw.plugin.json  # Plugin metadata
  index.ts              # Entry point, exports OpenClawPluginDefinition
  src/                  # Implementation code
```

## Provider System (src/providers/)

Model providers implement authentication and configuration for LLM APIs:

```typescript
type ProviderPlugin = {
  id: string;           // e.g., "anthropic"
  label: string;        // e.g., "Anthropic"
  aliases?: string[];
  envVars?: string[];   // e.g., ["ANTHROPIC_API_KEY"]
  models?: ModelProviderConfig;
  auth: ProviderAuthMethod[];
  formatApiKey?(cred): string;
  refreshOAuth?(cred): Promise<OAuthCredential>;
};
```

## Configuration System (src/config/)

- **Main config file**: `~/.openclaw/openclaw.json` (JSON5 format)
- **Schema**: Zod-based validation (`src/config/zod-schema.ts` and sub-files)
- **Types**: `src/config/types.ts` re-exports from `types.base.ts`, `types.channels.ts`, `types.discord.ts`, etc.
- **Paths**: `src/config/paths.ts` - Standard directory resolution
- **Runtime overrides**: `src/config/runtime-overrides.ts` - Environment-based overrides
- **Hot reload**: `src/gateway/config-reload.ts` - Config changes without restart

## Media Pipeline (src/media/)

- Image processing via `sharp`
- Audio transcription via `src/media-understanding/`
- PDF processing via `pdfjs-dist`
- Video frame extraction
- MIME type detection (`src/media/mime.ts`)
- Media storage (`src/media/store.ts`)

## Session Management (src/sessions/)

Sessions track conversation state:
- Session key: `{channelId}:{accountId}:{conversationId}`
- Persistence: File-based session storage
- Compaction: Automatic summarization when context grows large
- Lane system: Message queuing per session to prevent concurrent processing

## Security (src/security/)

- File permission auditing (`audit.ts`, `audit-fs.ts`)
- External content sanitization (`external-content.ts`)
- Windows ACL support (`windows-acl.ts`)
- Per-channel allowlist/blocklist enforcement
- Sender authorization checks

## Build System

- **TypeScript compilation**: `tsc` to `dist/`
- **Bundling**: rolldown for canvas A2UI
- **Linting**: oxlint (type-aware)
- **Formatting**: oxfmt
- **Testing**: vitest (parallel test runner via `scripts/test-parallel.mjs`)
- **Package manager**: pnpm 10.x with workspaces
- **Protocol generation**: `scripts/protocol-gen.ts` generates JSON schema + Swift types
