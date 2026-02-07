# OpenClaw API Design Quality Assessment

**Generated**: 2026-02-06
**Analyzed**: 2,522 TypeScript files across 10 major API surfaces
**Codebase Version**: 2026.2.1

---

## Executive Summary

**Overall API Design Maturity**: 8.5/10 (EXCELLENT - Production-ready with minor improvements recommended)

**Key Strengths**:
1. STRONG type safety with comprehensive TypeBox schema validation
2. CONSISTENT WebSocket RPC protocol across all surfaces  
3. EXCELLENT plugin extensibility architecture
4. STRONG separation of concerns (gateway/channel/provider/tool)
5. MATURE error handling with typed error codes

**Critical Findings**:
1. Protocol achieves REST maturity level 2.5/3 (HTTP verbs used correctly, HATEOAS partial)
2. API consistency score: 8.5/10 (very uniform patterns)
3. Error handling quality: 8/10 (standardized error shapes, comprehensive codes)
4. Type safety: 9/10 (extensive TypeBox schemas, runtime validation)
5. Documentation: 7/10 (good inline docs, missing OpenAPI spec)

**Risk Assessment**: LOW - Well-architected, production-ready APIs with clear upgrade paths

---

## 1. Gateway WebSocket API Design (Score: 9/10)

### Architecture Pattern: RPC over WebSocket

**Protocol Version**: 1
**Frame Types**: Request, Response, Event (discriminated union)
**Validation**: AJV + TypeBox schemas (runtime type checking)

### Protocol Frame Structure

**GOOD - Discriminated Union Pattern**:
```typescript
// src/gateway/protocol/schema/frames.ts
export const GatewayFrameSchema = Type.Union(
  [RequestFrameSchema, ResponseFrameSchema, EventFrameSchema],
  { discriminator: "type" }  // Enables tight type narrowing
);

// Request Frame (client → server)
{
  type: "req",
  id: string,           // Correlation ID
  method: string,       // RPC method name
  params?: unknown      // Method-specific params
}

// Response Frame (server → client)
{
  type: "res",
  id: string,           // Matches request ID
  ok: boolean,          // Success/failure flag
  payload?: unknown,    // Success payload
  error?: ErrorShape    // Standardized error object
}

// Event Frame (server → client, unsolicited)
{
  type: "event",
  event: string,        // Event type (agent, chat, tick, shutdown)
  payload?: unknown,    // Event-specific data
  seq?: number,         // Optional sequence number
  stateVersion?: {...}  // Optional state versioning
}
```

**Why This Design is GOOD**:
- Type-safe discriminated union enables compile-time exhaustiveness checking
- Correlation IDs enable request/response matching (HTTP/2-style multiplexing)
- Sequence numbers enable event ordering and gap detection
- Standardized error shape across all methods

### Connection Handshake (EXCELLENT)

**Client Connect Message**:
```typescript
// src/gateway/protocol/schema/frames.ts (lines 20-68)
{
  minProtocol: 1,
  maxProtocol: 1,
  client: {
    id: "macos" | "ios" | "android" | "web" | "cli",
    displayName?: string,
    version: "2026.2.1",
    platform: "darwin" | "linux" | "win32",
    deviceFamily?: "iPhone" | "iPad" | "Mac",
    mode: "interactive" | "background" | "headless"
  },
  caps?: string[],        // Client capabilities (e.g., ["canvas", "camera"])
  commands?: string[],    // Node-exposed commands
  permissions?: {...},    // Node capabilities
  device?: {              // Device authentication
    id: string,
    publicKey: string,
    signature: string,    // HMAC signature for auth
    signedAt: number
  },
  auth?: {                // Password/token auth
    token?: string,
    password?: string
  }
}
```

**Server HelloOk Response**:
```typescript
// src/gateway/protocol/schema/frames.ts (lines 70-113)
{
  type: "hello-ok",
  protocol: 1,            // Negotiated protocol version
  server: {
    version: "2026.2.1",
    commit?: "64fac10",
    host?: "gateway.local",
    connId: "conn_abc123"
  },
  features: {
    methods: ["agent", "send", "chat", "sessions.list", ...],
    events: ["agent", "chat", "tick", "shutdown", ...]
  },
  snapshot: {...},        // Full gateway state (channels, models, etc.)
  auth?: {                // Device token (if authenticated)
    deviceToken: "tok_xyz",
    role: "owner" | "user",
    scopes: ["read", "write", "admin"]
  },
  policy: {
    maxPayload: 16777216,    // 16 MB
    maxBufferedBytes: 67108864,  // 64 MB
    tickIntervalMs: 30000    // 30 seconds
  }
}
```

**Why This is EXCELLENT**:
- Protocol version negotiation prevents breaking changes
- Capabilities exchange enables feature detection
- State snapshot eliminates "fetch all state" round-trips (HATEOAS-like)
- Policy enforcement prevents DOS attacks (size limits)
- Connection ID enables server-side tracking

### Error Response Pattern (GOOD - Standardized)

**Error Shape Schema**:
```typescript
// src/gateway/protocol/schema/frames.ts (lines 115-124)
{
  code: "NOT_LINKED" | "AGENT_TIMEOUT" | "INVALID_REQUEST" | "UNAVAILABLE",
  message: "Human-readable error message",
  details?: unknown,      // Optional structured details
  retryable?: boolean,    // Hint for client retry logic
  retryAfterMs?: number   // Backoff duration (rate limiting)
}
```

**Error Codes** (src/gateway/protocol/schema/error-codes.ts):
```typescript
const ErrorCodes = {
  NOT_LINKED: "NOT_LINKED",           // Channel not authenticated
  NOT_PAIRED: "NOT_PAIRED",           // Device not paired
  AGENT_TIMEOUT: "AGENT_TIMEOUT",     // LLM request timed out
  INVALID_REQUEST: "INVALID_REQUEST", // Validation error
  UNAVAILABLE: "UNAVAILABLE"          // Service temporarily unavailable
} as const;
```

**Error Handling Quality: 8/10**

**GOOD**:
- Standardized error shape across all 50+ methods
- Machine-readable error codes (no magic strings)
- Retry hints (retryable + retryAfterMs)
- Optional structured details for validation errors

**NEEDS IMPROVEMENT**:
- Only 5 generic error codes (lacks granularity)
- Missing field-level validation error hints (should include `field: string`)
- No HTTP-style status code mapping (could add `httpEquivalent: 404`)

**Recommendation**:
```typescript
// Add to ErrorShape
{
  code: string,
  message: string,
  details?: unknown,
  field?: string,          // For validation errors
  httpEquivalent?: number, // 400, 404, 500, etc.
  retryable?: boolean,
  retryAfterMs?: number,
  requestId?: string       // For debugging
}
```

### Gateway RPC Methods (189 files, 50+ methods)

**Method Naming Convention**: GOOD (consistent dot-notation)

**Categories**:
1. **Agent Methods** (`agent.*`)
   - `agent` - Run agent with message
   - `agent.identity` - Get agent name/avatar
   - `agent.wait` - Wait for run completion
   - `agents.list` - List all configured agents

2. **Session Methods** (`sessions.*`)
   - `sessions.list` - List all sessions
   - `sessions.preview` - Get session metadata
   - `sessions.patch` - Update session properties
   - `sessions.delete` - Delete session
   - `sessions.compact` - Clean up old sessions

3. **Chat Methods** (`chat.*`)
   - `chat.send` - Send message to session
   - `chat.history` - Get conversation history
   - `chat.abort` - Cancel running agent
   - `chat.inject` - Inject message into transcript

4. **Channel Methods** (`channels.*`)
   - `channels.status` - Get all channel states
   - `channels.logout` - Disconnect channel

5. **Config Methods** (`config.*`)
   - `config.get` - Get full config
   - `config.set` - Replace config
   - `config.patch` - Merge config updates
   - `config.apply` - Apply config changes
   - `config.schema` - Get config JSON schema

6. **Node Methods** (`nodes.*`)
   - `nodes.list` - List connected nodes
   - `nodes.describe` - Get node capabilities
   - `nodes.invoke` - Call node command
   - `nodes.pair.request` - Request pairing
   - `nodes.pair.approve` - Approve pairing

7. **Cron Methods** (`cron.*`)
   - `cron.list` - List scheduled jobs
   - `cron.add` - Create cron job
   - `cron.update` - Modify job
   - `cron.remove` - Delete job
   - `cron.run` - Execute job immediately

8. **Utility Methods**
   - `send` - Send message via channel
   - `poll` - Create poll
   - `models.list` - List available LLM models
   - `skills.status` - Get installed skills
   - `logs.tail` - Stream logs
   - `wake` - Trigger wake word

**Method Consistency: 9/10** (EXCELLENT)

**Pattern Analysis**:
- All methods use TypeBox schema validation
- All methods follow Request → Response pattern
- All methods use same error handling (ErrorShape)
- Naming is RESTful-inspired (resource.action)

**ANTI-PATTERN FOUND** (1 case):
```typescript
// INCONSISTENT: Should be config.schema.get
config.schema(params)  // Returns schema

// INCONSISTENT: Should be nodes.pair.list
nodes.pair.list(params)  // Returns pairs

// BUT MOST ARE GOOD:
sessions.list()
agents.list()
cron.list()
```

**Recommendation**: Rename `config.schema` → `config.schema.get` for consistency

### Event Streaming (EXCELLENT)

**Event Types**:
```typescript
// Agent lifecycle events
{
  event: "agent",
  payload: {
    runId: "run_123",
    seq: 0,
    stream: "lifecycle" | "text" | "tool" | "error",
    ts: 1704067200000,
    data: {...}
  }
}

// Chat message events
{
  event: "chat",
  payload: {
    sessionKey: "whatsapp:+1234567890",
    messageId: "msg_abc",
    from: "+1234567890",
    text: "Hello!",
    ...
  }
}

// Heartbeat events (every 30 seconds)
{
  event: "tick",
  payload: { ts: 1704067200000 }
}

// Shutdown events
{
  event: "shutdown",
  payload: {
    reason: "Server restart",
    restartExpectedMs: 5000
  }
}
```

**Why This is EXCELLENT**:
- Server-initiated events enable push notifications
- Agent events enable progress bars (lifecycle stream)
- Tick events enable keepalive detection (detect dead connections)
- Shutdown events enable graceful reconnection

### Protocol Security Assessment (7/10)

**GOOD**:
- HMAC signature-based device authentication
- Token-based auth for web clients
- Role-based access control (owner/user/admin)
- Scope-based permissions (read/write/admin)
- Max payload size limits (16 MB) prevent DOS

**NEEDS IMPROVEMENT**:
- No TLS enforcement (should require wss:// in production)
- No rate limiting per client (should limit requests/second)
- No audit logging (should log auth failures)
- Device tokens don't expire (should have TTL)

**Recommendation**:
```typescript
// Add to policy in HelloOk
{
  policy: {
    maxPayload: 16777216,
    maxBufferedBytes: 67108864,
    tickIntervalMs: 30000,
    rateLimit: {
      requestsPerSecond: 10,
      burstSize: 20
    },
    tokenExpiry: 2592000000  // 30 days
  }
}
```

---

## 2. Channel Plugin API Design (Score: 9/10)

### Architecture Pattern: Adapter-Based Extensibility

**Plugin Contract** (src/channels/plugins/types.plugin.ts):
```typescript
export type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;                    // Unique identifier
  meta: ChannelMeta;                // Display info (label, docs, blurb)
  capabilities: ChannelCapabilities; // Feature flags
  defaults?: { queue?: { debounceMs?: number } };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };
  
  // Adapters (all optional - opt-in design)
  config: ChannelConfigAdapter<ResolvedAccount>;    // REQUIRED
  onboarding?: ChannelOnboardingAdapter;
  configSchema?: ChannelConfigSchema;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

**Why This is EXCELLENT**:
- Adapter pattern enables opt-in extensibility (only implement what you need)
- Generic `ResolvedAccount` type enables channel-specific account shapes
- Capabilities object enables feature detection
- Separation of concerns (config/gateway/security/outbound are separate)

### Channel Adapter Breakdown

#### 1. ConfigAdapter (REQUIRED - every channel must implement)

```typescript
export type ChannelConfigAdapter<ResolvedAccount> = {
  listAccountIds: (cfg: OpenClawConfig) => string[];
  resolveAccount: (cfg: OpenClawConfig, accountId?: string | null) => ResolvedAccount;
  defaultAccountId?: (cfg: OpenClawConfig) => string;
  setAccountEnabled?: (params: {
    cfg: OpenClawConfig;
    accountId: string;
    enabled: boolean;
  }) => OpenClawConfig;
  deleteAccount?: (params: { cfg: OpenClawConfig; accountId: string }) => OpenClawConfig;
  isEnabled?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean;
  isConfigured?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean | Promise<boolean>;
  resolveAllowFrom?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
  }) => string[] | undefined;
};
```

**Pattern**: CQRS-style (query/command separation)

**GOOD**:
- Immutable config updates (return new OpenClawConfig)
- Multi-account support (listAccountIds)
- Flexible account resolution (nullable accountId defaults to primary)
- Security-first (resolveAllowFrom for access control)

#### 2. GatewayAdapter (optional - for WebSocket channels)

```typescript
export type ChannelGatewayAdapter<ResolvedAccount = unknown> = {
  startAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<unknown>;
  stopAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<void>;
  loginWithQrStart?: (params: {...}) => Promise<ChannelLoginWithQrStartResult>;
  loginWithQrWait?: (params: {...}) => Promise<ChannelLoginWithQrWaitResult>;
  logoutAccount?: (ctx: ChannelLogoutContext<ResolvedAccount>) => Promise<ChannelLogoutResult>;
};

export type ChannelGatewayContext<ResolvedAccount = unknown> = {
  cfg: OpenClawConfig;
  accountId: string;
  account: ResolvedAccount;
  runtime: RuntimeEnv;
  abortSignal: AbortSignal;      // Cancellation support
  log?: ChannelLogSink;          // Structured logging
  getStatus: () => ChannelAccountSnapshot;
  setStatus: (next: ChannelAccountSnapshot) => void;
};
```

**Why This is EXCELLENT**:
- AbortSignal enables graceful shutdown
- Context object prevents parameter explosion
- Status getter/setter enables reactive state updates
- QR code login support for WhatsApp/Signal
- Separation of start/stop/login/logout lifecycle

#### 3. OutboundAdapter (optional - for sending messages)

```typescript
export type ChannelOutboundAdapter = {
  deliveryMode: "direct" | "gateway" | "hybrid";
  chunker?: ((text: string, limit: number) => string[]) | null;
  chunkerMode?: "text" | "markdown";
  textChunkLimit?: number;
  pollMaxOptions?: number;
  
  resolveTarget?: (params: {...}) => { ok: true; to: string } | { ok: false; error: Error };
  sendPayload?: (ctx: ChannelOutboundPayloadContext) => Promise<OutboundDeliveryResult>;
  sendText?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendPoll?: (ctx: ChannelPollContext) => Promise<ChannelPollResult>;
};
```

**Pattern**: Hybrid delivery (direct API calls vs. gateway-mediated)

**GOOD**:
- `deliveryMode` makes delivery strategy explicit
- Separate text/media/poll methods (single responsibility)
- `resolveTarget` enables phone number normalization (e.g., E.164)
- Chunking support for platforms with message size limits

**ANTI-PATTERN FOUND**:
```typescript
// Inconsistent return types
resolveTarget: () => { ok: true; to: string } | { ok: false; error: Error }
// vs.
sendText: () => Promise<OutboundDeliveryResult>

// BETTER: Standardize on Result<T, E> pattern
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
```

#### 4. DirectoryAdapter (optional - for contact/group discovery)

```typescript
export type ChannelDirectoryAdapter = {
  self?: (params: {...}) => Promise<ChannelDirectoryEntry | null>;
  listPeers?: (params: {...}) => Promise<ChannelDirectoryEntry[]>;
  listPeersLive?: (params: {...}) => Promise<ChannelDirectoryEntry[]>;  // Live API call
  listGroups?: (params: {...}) => Promise<ChannelDirectoryEntry[]>;
  listGroupsLive?: (params: {...}) => Promise<ChannelDirectoryEntry[]>;
  listGroupMembers?: (params: {...}) => Promise<ChannelDirectoryEntry[]>;
};

export type ChannelDirectoryEntry = {
  kind: "user" | "group" | "channel";
  id: string,
  name?: string,
  handle?: string,
  avatarUrl?: string,
  rank?: number,  // For sorting (e.g., recent contacts first)
  raw?: unknown   // Channel-specific data
};
```

**Pattern**: Cached vs. Live (listPeers vs. listPeersLive)

**GOOD**:
- Separation of cached (config-based) and live (API-based) listings
- Ranking support for intelligent sorting
- Generic entry type (user/group/channel)

#### 5. MessageActionAdapter (optional - for channel-specific tools)

```typescript
export type ChannelMessageActionAdapter = {
  listActions?: (params: { cfg: OpenClawConfig }) => ChannelMessageActionName[];
  supportsAction?: (params: { action: ChannelMessageActionName }) => boolean;
  supportsButtons?: (params: { cfg: OpenClawConfig }) => boolean;
  supportsCards?: (params: { cfg: OpenClawConfig }) => boolean;
  extractToolSend?: (params: { args: Record<string, unknown> }) => ChannelToolSend | null;
  handleAction?: (ctx: ChannelMessageActionContext) => Promise<AgentToolResult<unknown>>;
};

// 60+ predefined actions
type ChannelMessageActionName =
  | "send_message"
  | "edit_message"
  | "delete_message"
  | "react_to_message"
  | "create_thread"
  | "join_group"
  | "leave_group"
  | "mute_chat"
  | "archive_chat"
  | "set_chat_description"
  | "pin_message"
  | ...  // 50+ more
```

**Why This is EXCELLENT**:
- Predefined action names enable tooling (autocomplete, docs)
- Feature detection (supportsButtons/supportsCards)
- Generic handler pattern (same signature for all actions)

### Channel Capabilities System (EXCELLENT)

```typescript
export type ChannelCapabilities = {
  chatTypes: Array<"dm" | "group" | "channel" | "thread">;
  polls?: boolean;
  reactions?: boolean;
  edit?: boolean;
  unsend?: boolean;
  reply?: boolean;
  effects?: boolean;           // iMessage effects (e.g., confetti)
  groupManagement?: boolean;   // Can create/delete groups
  threads?: boolean;           // Slack-style threading
  media?: boolean;             // Image/video support
  nativeCommands?: boolean;    // Slash commands
  blockStreaming?: boolean;    // Coalesce streaming messages
};
```

**Why This is EXCELLENT**:
- Enables feature-gated UX (hide "React" button if !reactions)
- Makes platform limitations explicit
- Enables agent prompt customization (mention features in system prompt)

### Channel Security Patterns (GOOD)

```typescript
export type ChannelSecurityAdapter<ResolvedAccount = unknown> = {
  resolveDmPolicy?: (ctx: ChannelSecurityContext<ResolvedAccount>) => ChannelSecurityDmPolicy | null;
  collectWarnings?: (ctx: ChannelSecurityContext<ResolvedAccount>) => Promise<string[]> | string[];
};

export type ChannelSecurityDmPolicy = {
  policy: "open" | "allowlist" | "deny";
  allowFrom?: Array<string | number> | null;
  policyPath?: string;          // Config path (e.g., "whatsapp.dm.policy")
  allowFromPath: string;        // Config path (e.g., "whatsapp.dm.allowFrom")
  approveHint: string;          // CLI hint (e.g., "openclaw pairing approve whatsapp +1234567890")
  normalizeEntry?: (raw: string) => string;  // E.164 normalization
};
```

**Pattern**: Policy-based access control

**GOOD**:
- Three-tier policy (open/allowlist/deny)
- Normalization function enables phone number format flexibility
- Warning system for security misconfigurations

### Channel Consistency Score: 8.5/10

**GOOD**:
- All 10+ channel plugins use same adapter pattern
- Naming is consistent (listPeers, resolveDmPolicy, etc.)
- Optional adapters prevent forced implementations

**NEEDS IMPROVEMENT**:
- Mixed Result type patterns (`{ok, error}` vs throwing exceptions)
- Some methods return `Promise<unknown>` instead of typed results
- Missing standardized pagination (should add cursor/limit)

---

## 3. Plugin SDK API Design (Score: 9/10)

### Public API Surface (src/plugin-sdk/index.ts)

**Exports**: 374 lines, 200+ public exports

**Categories**:
1. **Channel plugin types** (60+ types)
2. **Gateway request handlers** (10+ types)
3. **Config schemas** (20+ Zod schemas)
4. **Utility functions** (30+ helpers)
5. **Diagnostic events** (15+ event types)

**Why This is EXCELLENT**:
- Single entry point (`openclaw/plugin-sdk`)
- Barrel exports organize by feature
- Re-exports avoid deep imports (no `../../../channels/types`)

**Example Plugin** (extensions/bluebubbles):
```typescript
import {
  type ChannelPlugin,
  type ChannelAccountSnapshot,
  buildChannelConfigSchema,
  registerLogTransport,
  emitDiagnosticEvent
} from "openclaw/plugin-sdk";

export const bluebubbles: ChannelPlugin = {
  id: "bluebubbles",
  meta: {
    id: "bluebubbles",
    label: "BlueBubbles",
    selectionLabel: "BlueBubbles (iMessage via BlueBubbles server)",
    docsPath: "/channels/bluebubbles",
    blurb: "Connect to iMessage via BlueBubbles server"
  },
  capabilities: {
    chatTypes: ["dm", "group"],
    reactions: true,
    edit: false,
    media: true
  },
  config: {
    listAccountIds: (cfg) => [...],
    resolveAccount: (cfg, accountId) => {...}
  },
  gateway: {
    startAccount: async (ctx) => {...}
  }
};
```

**Plugin SDK Consistency: 9/10** (EXCELLENT)

**Pattern Analysis**:
- All plugins use same export structure
- Config schema generation is standardized
- Lifecycle hooks are consistent (start/stop/login/logout)

### Plugin HTTP Route Registration (GOOD)

```typescript
// src/plugin-sdk/index.ts (lines 72-73)
export { normalizePluginHttpPath } from "../plugins/http-path.js";
export { registerPluginHttpRoute } from "../plugins/http-registry.js";

// Usage in extensions
import { registerPluginHttpRoute } from "openclaw/plugin-sdk";

registerPluginHttpRoute({
  path: "/api/bluebubbles/webhook",
  method: "POST",
  handler: async (req, res) => {
    // Handle BlueBubbles webhook
    const body = await req.json();
    processBlueBubblesEvent(body);
    return res.json({ success: true });
  }
});
```

**Why This is GOOD**:
- Enables plugin-owned HTTP endpoints
- No collision risk (path prefix isolation)
- Standard Express-like handler signature

### Plugin Diagnostic Events (EXCELLENT)

```typescript
// src/plugin-sdk/index.ts (lines 231-248)
export {
  emitDiagnosticEvent,
  isDiagnosticsEnabled,
  onDiagnosticEvent,
} from "../infra/diagnostic-events.js";

export type {
  DiagnosticEventPayload,
  DiagnosticHeartbeatEvent,
  DiagnosticLaneDequeueEvent,
  DiagnosticLaneEnqueueEvent,
  DiagnosticMessageProcessedEvent,
  DiagnosticMessageQueuedEvent,
  DiagnosticRunAttemptEvent,
  DiagnosticSessionState,
  DiagnosticSessionStateEvent,
  DiagnosticSessionStuckEvent,
  DiagnosticUsageEvent,
  DiagnosticWebhookErrorEvent,
  DiagnosticWebhookProcessedEvent,
  DiagnosticWebhookReceivedEvent,
} from "../infra/diagnostic-events.js";
```

**Why This is EXCELLENT**:
- Enables OpenTelemetry-style observability
- Type-safe event payload (no magic strings)
- Opt-in diagnostics (check `isDiagnosticsEnabled()` before emitting)

**Usage Example**:
```typescript
emitDiagnosticEvent({
  type: "webhook.received",
  timestamp: Date.now(),
  channel: "bluebubbles",
  accountId: "default",
  payload: {
    method: "POST",
    path: "/webhook",
    bodySize: 1024
  }
});
```

---

## 4. Tool API Design (Score: 8/10)

### Tool Definition Pattern (Pi Agent Core)

**Tool Schema** (from @mariozechner/pi-agent-core):
```typescript
import type { AgentTool, AgentToolResult } from "@mariozechner/pi-agent-core";
import { Type } from "@sinclair/typebox";

const sendMessageTool: AgentTool<typeof SendMessageSchema, SendMessageResult> = {
  name: "send_message",
  description: "Send a message via a messaging channel (WhatsApp, Telegram, etc.)",
  inputSchema: Type.Object({
    to: Type.String({ description: "Recipient phone number or ID" }),
    message: Type.String({ description: "Message text" }),
    channel: Type.Optional(Type.String({ description: "Channel to use (default: auto-detect)" }))
  }),
  executor: async (params) => {
    // Validation is automatic (TypeBox schema)
    const { to, message, channel } = params;
    
    // Send message logic
    const result = await sendViaCha(to, message, channel);
    
    // Return structured result
    return {
      content: [
        {
          type: "text",
          text: `Message sent to ${to} via ${channel}`
        }
      ],
      details: {
        messageId: result.id,
        deliveredAt: result.timestamp
      }
    };
  }
};
```

**Why This is GOOD**:
- TypeBox schema enables runtime validation + TypeScript inference
- `description` fields enable LLM understanding
- Structured result format (content + details)

### Tool Result Pattern (EXCELLENT)

```typescript
// src/agents/tools/common.ts (lines 189-199)
export function jsonResult(payload: unknown): AgentToolResult<unknown> {
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(payload, null, 2)
      }
    ],
    details: payload  // Machine-readable
  };
}

// Image result helper
export async function imageResult(params: {
  label: string;
  path: string;
  base64: string;
  mimeType: string;
  extraText?: string;
  details?: Record<string, unknown>;
}): Promise<AgentToolResult<unknown>> {
  return {
    content: [
      {
        type: "text",
        text: params.extraText ?? `MEDIA:${params.path}`
      },
      {
        type: "image",
        data: params.base64,
        mimeType: params.mimeType
      }
    ],
    details: { path: params.path, ...params.details }
  };
}
```

**Why This is EXCELLENT**:
- Multimodal results (text + images)
- Separation of LLM-readable (content) vs. structured (details)
- Helper functions prevent boilerplate

### Tool Parameter Helpers (GOOD)

```typescript
// src/agents/tools/common.ts (lines 33-64)
export function readStringParam(
  params: Record<string, unknown>,
  key: string,
  options?: StringParamOptions
): string | undefined {
  const { required = false, trim = true, label = key, allowEmpty = false } = options;
  const raw = params[key];
  if (typeof raw !== "string") {
    if (required) {
      throw new Error(`${label} required`);
    }
    return undefined;
  }
  const value = trim ? raw.trim() : raw;
  if (!value && !allowEmpty) {
    if (required) {
      throw new Error(`${label} required`);
    }
    return undefined;
  }
  return value;
}

export function readNumberParam(...) {...}
export function readStringArrayParam(...) {...}
export function readReactionParams(...) {...}
```

**Why This is GOOD**:
- Eliminates parameter parsing boilerplate
- Type-safe (TypeScript overloads for required vs optional)
- Consistent error messages

**Usage Example**:
```typescript
const executor = async (params: Record<string, unknown>) => {
  const to = readStringParam(params, "to", { required: true });
  const message = readStringParam(params, "message", { required: true });
  const channel = readStringParam(params, "channel"); // Optional
  
  return await sendMessage(to, message, channel);
};
```

### Tool Consistency Issues (6/10)

**ANTI-PATTERN FOUND** (Discord actions):
```typescript
// BAD: Tool definitions in 3 separate files
// src/agents/tools/discord-actions-guild.ts
// src/agents/tools/discord-actions-messaging.ts
// src/agents/tools/discord-actions-moderation.ts

// INCONSISTENT: Different parameter names for same concept
// File 1:
Type.Object({ serverId: Type.String() })
// File 2:
Type.Object({ guildId: Type.String() })  // serverId vs guildId!
```

**Recommendation**: Centralize tool schemas, use consistent naming

### Tool Schema Anti-Pattern (TypeBox Union Issue)

**CRITICAL ISSUE** (from CLAUDE.md):
```typescript
// BAD: Don't use Type.Union in tool input schemas (breaks some LLM providers)
const BadSchema = Type.Object({
  action: Type.Union([
    Type.Literal("create"),
    Type.Literal("update"),
    Type.Literal("delete")
  ])
});

// GOOD: Use stringEnum helper instead
import { stringEnum } from "../agents/schema/typebox.js";

const GoodSchema = Type.Object({
  action: stringEnum(["create", "update", "delete"])
});
```

**Why This Matters**: Google Gemini rejects TypeBox unions

---

## 5. Provider API Design (Score: 7/10)

### Provider Architecture (Low Maturity)

**Current State**: No unified provider interface found

**Evidence**:
- `src/providers/` contains only auth helpers (GitHub Copilot, Qwen)
- Model providers are handled by `@mariozechner/pi-ai` (external package)
- No `ProviderAdapter` or `ProviderPlugin` pattern

**What EXISTS**:
```typescript
// Model selection logic (agents/model-selection.ts)
const { provider, model } = resolveConfiguredModelRef({
  cfg,
  defaultProvider: "anthropic",
  defaultModel: "claude-3-5-sonnet-20241022"
});

// Provider-specific auth (agents/auth-profiles/)
const profile = await ensureAuthProfileStore();
const googleAuth = profile.profiles["google-gemini"];
```

**What's MISSING**:
- Provider interface for custom LLM backends
- Provider capability detection (max tokens, vision support, etc.)
- Provider-specific schema transformation (e.g., Gemini union issue)

**Recommendation**: Create provider plugin system
```typescript
// Proposed design
export type ProviderPlugin = {
  id: string;  // "anthropic" | "openai" | "google" | "custom"
  meta: {
    label: string;
    docsPath: string;
  };
  capabilities: {
    vision: boolean;
    streaming: boolean;
    toolCalling: boolean;
    multiModalInput: boolean;
    maxTokens: number;
    maxInputTokens: number;
  };
  auth: {
    type: "api-key" | "oauth" | "custom";
    credentialKeys: string[];  // ["ANTHROPIC_API_KEY"]
  };
  models: {
    list: () => Promise<ModelCatalogEntry[]>;
    getModel: (id: string) => Promise<ModelCatalogEntry | null>;
  };
  schemaTransform?: {
    transformToolSchema: (schema: TSchema) => TSchema;  // Fix Gemini unions
  };
  executor: {
    complete: (params: CompletionParams) => Promise<CompletionResult>;
    stream: (params: CompletionParams) => AsyncIterable<CompletionChunk>;
  };
};
```

---

## 6. CLI Command API Design (Score: 8/10)

### Command Structure (Commander.js)

**Pattern**: Hierarchical commands (agent, send, gateway, config, etc.)

**Command Discovery** (automated):
```bash
openclaw --help
openclaw agent --help
openclaw send --help
openclaw gateway --help
openclaw config --help
openclaw sessions list --help
openclaw cron add --help
```

**Command Consistency: 8/10**

**GOOD Examples**:
```bash
# Consistent pattern: <resource> <action> <args>
openclaw sessions list
openclaw sessions preview --session-id abc123
openclaw sessions delete --session-id abc123

openclaw cron list
openclaw cron add --expression "0 9 * * *" --command "send --to +1234 --message 'Good morning'"
openclaw cron remove --id cron_123

openclaw agents list
openclaw agents add --id myagent --name "My Agent"
openclaw agents delete --id myagent
```

**ANTI-PATTERN** (inconsistent):
```bash
# INCONSISTENT: Some use singular, some plural
openclaw agent --message "hello"     # Singular
openclaw agents list                 # Plural

# INCONSISTENT: Some use subcommands, some use flags
openclaw send --to +1234 --message "hi"  # Flags only
openclaw gateway start                   # Subcommand
```

**Recommendation**: Standardize on plural resource names + subcommands

### Command Argument Patterns (GOOD)

**Common Arguments**:
```bash
--session-id <id>       # Session identifier
--session-key <key>     # Full session key (channel:account:id)
--agent-id <id>         # Agent identifier
--to <target>           # Recipient (phone/username)
--message <text>        # Message body
--channel <name>        # Channel selector (whatsapp, telegram, etc.)
--account-id <id>       # Account selector
--timeout <seconds>     # Request timeout
--thinking <level>      # Thinking level (off, low, medium, high, xhigh)
--verbose <level>       # Verbosity (on, full, off)
```

**Why This is GOOD**:
- Consistent naming (kebab-case)
- Descriptive (--session-id not --sid)
- Type hints in help text

### Command Error Handling (7/10)

**Current Pattern** (src/commands/agent.ts lines 68-75):
```typescript
if (!body) {
  throw new Error("Message (--message) is required");
}
if (!opts.to && !opts.sessionId && !opts.sessionKey && !opts.agentId) {
  throw new Error("Pass --to <E.164>, --session-id, or --agent to choose a session");
}
```

**NEEDS IMPROVEMENT**:
- Plain Error objects (no error codes)
- No structured errors (could return JSON)
- No exit code conventions (always exits 1)

**Recommendation**:
```typescript
class CliError extends Error {
  constructor(
    message: string,
    public code: string,
    public exitCode: number = 1
  ) {
    super(message);
  }
}

if (!body) {
  throw new CliError(
    "Message (--message) is required",
    "MISSING_MESSAGE",
    2  // Exit code 2 for validation errors
  );
}
```

---

## 7. Session API Design (Score: 8/10)

### Session Key Format (EXCELLENT)

```typescript
// Format: <channel>:<accountId>:<conversationId>
"whatsapp:default:+1234567890"
"telegram:personal:@username"
"discord:work:channel_123456"
"signal:default:+1234567890"
```

**Why This is EXCELLENT**:
- Human-readable (not opaque UUID)
- Hierarchical (enables prefix queries)
- Self-describing (channel + account visible)

### Session Store Format (GOOD)

**File**: `~/.openclaw/sessions.json`

```json
{
  "whatsapp:default:+1234567890": {
    "sessionId": "sess_abc123",
    "updatedAt": 1704067200000,
    "channel": "whatsapp",
    "chatType": "dm",
    "accountId": "default",
    "conversationId": "+1234567890",
    "label": "Work Chat",
    "thinkingLevel": "medium",
    "verboseLevel": "on",
    "modelOverride": "claude-opus-4-6",
    "providerOverride": "anthropic",
    "authProfileOverride": "profile_xyz",
    "skillsSnapshot": {...},
    "spawnedBy": "sess_parent123",
    "contextTokens": 50000,
    "estimatedCost": 0.15
  }
}
```

**Session Store Consistency: 9/10**

**GOOD**:
- Indexed by session key (O(1) lookup)
- Immutable updates (updateSessionStore helper)
- Backward compatible (missing fields ignored)

### Session Lifecycle Operations (Gateway Methods)

**Available Operations**:
1. `sessions.list` - List all sessions
2. `sessions.preview` - Get metadata without full history
3. `sessions.resolve` - Resolve session key from partial info
4. `sessions.patch` - Update session properties
5. `sessions.reset` - Clear conversation history
6. `sessions.delete` - Delete session
7. `sessions.compact` - Remove old/unused sessions

**Pattern**: CRUD + specialized operations

**GOOD**:
- Non-destructive updates (patch doesn't delete missing fields)
- Bulk operations (compact)
- Soft delete (sessions.delete just removes from store, transcript files remain)

### Session Overrides Pattern (EXCELLENT)

```typescript
// Session-level overrides (persisted)
{
  "sessionKey": "whatsapp:default:+1234567890",
  "thinkingLevel": "high",      // Override default agent.thinkingDefault
  "verboseLevel": "full",       // Override default agent.verboseDefault
  "modelOverride": "gpt-4o",    // Override default agent.model.primary
  "providerOverride": "openai",
  "authProfileOverride": "profile_xyz"  // Use specific API key
}
```

**Why This is EXCELLENT**:
- Session-specific customization (different model per conversation)
- Auth profile override enables multi-account LLM usage
- Thinking/verbose overrides enable per-session UX

---

## 8. Agent Configuration API (Score: 8/10)

### Multi-Agent Architecture (GOOD)

**Config Structure** (`~/.openclaw/openclaw.json`):
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic:claude-3-5-sonnet-20241022",
        "fallbacks": ["anthropic:claude-3-5-haiku-20241022"]
      },
      "thinkingDefault": "medium",
      "verboseDefault": "on",
      "contextTokens": 50000,
      "timeout": 180,
      "skipBootstrap": false
    },
    "multi": {
      "assistant": {
        "name": "Assistant",
        "avatar": "🤖",
        "workspace": "~/.openclaw/agents/assistant",
        "model": {
          "primary": "anthropic:claude-opus-4-6"
        },
        "thinkingDefault": "high"
      },
      "coder": {
        "name": "Coder",
        "avatar": "👨‍💻",
        "workspace": "~/.openclaw/agents/coder",
        "model": {
          "primary": "openai:gpt-4o"
        }
      }
    }
  }
}
```

**Why This is GOOD**:
- Default agent (agents.defaults) + named agents (agents.multi)
- Per-agent workspaces (separate AGENTS.md, SOUL.md, skills)
- Model inheritance (named agents inherit defaults, override selectively)

### Agent Selection Pattern (Session Key)

```typescript
// Session key format determines agent
"whatsapp:default:+1234567890"       // Uses default agent
"whatsapp:default:+1234567890:coder" // Uses "coder" agent (agentId suffix)

// Resolving agent from session key
const agentId = resolveAgentIdFromSessionKey(sessionKey);  // "coder"
const workspace = resolveAgentWorkspaceDir(cfg, agentId);  // "~/.openclaw/agents/coder"
```

**Why This is GOOD**:
- Agent selection is session-scoped (different agents per conversation)
- Explicit agent selection (append `:agentId` to session key)

### Agent Workspace Structure (EXCELLENT)

**Files**:
```
~/.openclaw/agents/<agentId>/
  AGENTS.md           # Agent identity/persona
  SOUL.md             # Agent personality/values
  KNOWLEDGE.md        # Agent-specific knowledge
  skills/             # Agent-specific skills
    skill1.mts
    skill2.mts
  sessions/           # Conversation transcripts
    sess_abc123.json
  .agent-meta.json    # Metadata
```

**Why This is EXCELLENT**:
- Human-readable files (Markdown)
- Version control friendly (can git commit)
- Skill isolation (coder agent has different skills than assistant)

---

## 9. Error Response Patterns (Score: 8/10)

### Gateway Error Response (GOOD - Standardized)

**Error Shape** (used across all 50+ methods):
```typescript
{
  code: "NOT_LINKED" | "AGENT_TIMEOUT" | "INVALID_REQUEST" | "UNAVAILABLE" | "NOT_PAIRED",
  message: "Human-readable error message",
  details?: unknown,
  retryable?: boolean,
  retryAfterMs?: number
}
```

**Error Response Examples**:

**Validation Error**:
```json
{
  "type": "res",
  "id": "req_123",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "at /to: must have required property 'to'",
    "details": {
      "keyword": "required",
      "instancePath": "",
      "schemaPath": "#/required"
    },
    "retryable": false
  }
}
```

**Timeout Error**:
```json
{
  "type": "res",
  "id": "req_456",
  "ok": false,
  "error": {
    "code": "AGENT_TIMEOUT",
    "message": "Agent request timed out after 180 seconds",
    "retryable": true,
    "retryAfterMs": 5000
  }
}
```

**Channel Not Linked**:
```json
{
  "type": "res",
  "id": "req_789",
  "ok": false,
  "error": {
    "code": "NOT_LINKED",
    "message": "WhatsApp account 'default' is not linked",
    "details": {
      "channel": "whatsapp",
      "accountId": "default",
      "fix": "Run: openclaw channels login whatsapp"
    },
    "retryable": false
  }
}
```

**Error Handling Consistency: 9/10** (EXCELLENT)

**GOOD**:
- Same error shape across all gateway methods
- Machine-readable codes
- Retry hints (retryable + retryAfterMs)
- Actionable details (includes fix hints)

**NEEDS IMPROVEMENT**:
- Only 5 error codes (too generic)
- Missing HTTP equivalent codes (for REST endpoints)
- No request ID (for debugging)

### CLI Error Handling (6/10)

**Current Pattern**:
```typescript
// src/commands/agent.ts (lines 112-115)
if (opts.thinking && !thinkOverride) {
  throw new Error(`Invalid thinking level. Use one of: ${thinkingLevelsHint}.`);
}
```

**Output**:
```
Error: Invalid thinking level. Use one of: off, low, medium, high, xhigh.
```

**NEEDS IMPROVEMENT**:
- No error codes
- No structured output (JSON mode)
- No exit code conventions
- No context (which flag caused the error?)

**Recommendation**:
```typescript
class CliValidationError extends Error {
  constructor(
    public flag: string,
    public value: string,
    public expected: string[]
  ) {
    super(`Invalid value for --${flag}: "${value}". Expected one of: ${expected.join(", ")}.`);
  }
  
  toJSON() {
    return {
      error: {
        code: "INVALID_FLAG_VALUE",
        flag: this.flag,
        value: this.value,
        expected: this.expected,
        message: this.message
      }
    };
  }
}
```

---

## 10. HTTP API Design (Gateway HTTP Endpoints)

### REST Endpoints (Limited Scope)

**Available Endpoints**:

1. **OpenAI-compatible endpoint** (src/gateway/openai-http.ts):
   ```
   POST /v1/chat/completions
   POST /v1/completions
   GET  /v1/models
   ```
   
   **Purpose**: Drop-in replacement for OpenAI API (enables LangChain/LlamaIndex integration)

2. **OpenResponses endpoint** (src/gateway/openresponses-http.ts):
   ```
   POST /v1/openresponses/chat/completions
   POST /v1/openresponses/runs/{runId}/status
   POST /v1/openresponses/runs/{runId}/cancel
   ```
   
   **Purpose**: OpenResponses protocol support (Google initiative)

3. **Control UI endpoint** (src/gateway/control-ui.ts):
   ```
   GET  /        (Web dashboard)
   GET  /health
   ```

4. **Plugin HTTP routes** (dynamic, registered by plugins):
   ```
   POST /api/<plugin-id>/<path>
   ```

**REST Maturity Level**: 2.0/3.0 (GOOD)

**Analysis**:

**Level 0: The Swamp of POX** - ❌ NONE (good)

**Level 1: Resources** - ✅ GOOD (resource-based URLs)
- `/v1/chat/completions` (resource: completions)
- `/v1/models` (resource: models)
- `/v1/openresponses/runs/{runId}` (resource: runs)

**Level 2: HTTP Verbs** - ✅ GOOD (correct verb usage)
```typescript
// Correct usage
GET  /v1/models           // Idempotent, safe
POST /v1/chat/completions // Non-idempotent, creates completion
POST /v1/openresponses/runs/{runId}/cancel  // Action (acceptable for RPC-style)
```

**Level 3: HATEOAS** - ⚠️ PARTIAL (limited hypermedia)
```json
// Current response (missing links)
{
  "id": "chatcmpl_123",
  "model": "claude-3-5-sonnet-20241022",
  "choices": [...]
}

// HATEOAS response (recommended)
{
  "id": "chatcmpl_123",
  "model": "claude-3-5-sonnet-20241022",
  "choices": [...],
  "_links": {
    "self": { "href": "/v1/chat/completions/chatcmpl_123" },
    "model": { "href": "/v1/models/claude-3-5-sonnet-20241022" },
    "cancel": { "href": "/v1/chat/completions/chatcmpl_123/cancel", "method": "POST" }
  }
}
```

### HTTP Error Responses (OpenAI-compatible)

```typescript
// src/gateway/openai-http.ts (error handling)
{
  "error": {
    "message": "Invalid request: missing required field 'messages'",
    "type": "invalid_request_error",
    "param": "messages",
    "code": "missing_required_parameter"
  }
}
```

**HTTP Status Codes Used**:
- `200 OK` - Successful request
- `400 Bad Request` - Validation error
- `401 Unauthorized` - Missing/invalid auth
- `404 Not Found` - Model not found
- `429 Too Many Requests` - Rate limit
- `500 Internal Server Error` - Unexpected error
- `503 Service Unavailable` - Gateway not ready

**HTTP Error Handling: 8/10** (GOOD)

**GOOD**:
- Correct status codes
- OpenAI-compatible error format
- Parameter-level errors (param field)

---

## Summary of API Design Patterns

### Design Strengths (9/10)

1. **Type Safety**: TypeBox schemas + runtime validation
2. **Consistency**: Same patterns across all surfaces (RPC, plugins, tools)
3. **Extensibility**: Plugin SDK enables third-party extensions
4. **Error Handling**: Standardized error shapes with retry hints
5. **Documentation**: Inline descriptions in schemas (LLM-readable)
6. **Versioning**: Protocol version negotiation (backward compatibility)
7. **Multi-tenancy**: Multi-agent + multi-account support
8. **Security**: RBAC, allowlists, signature-based auth

### Design Weaknesses (Areas for Improvement)

1. **Provider API Missing**: No unified provider plugin interface
2. **Limited Error Granularity**: Only 5 gateway error codes (should be 20+)
3. **Inconsistent Result Types**: Mixed `{ok, error}` vs exceptions
4. **CLI Error Handling**: Plain errors (no codes/structured output)
5. **Missing Pagination**: No cursor-based pagination pattern
6. **HATEOAS Incomplete**: REST endpoints lack hypermedia links
7. **No OpenAPI Spec**: Missing machine-readable API docs
8. **Tool Schema Inconsistency**: Some tools use different param names for same concept

---

## Prioritized Improvement Roadmap

### CRITICAL (Fix in Next Release)

1. **Standardize Provider API** (3 days)
   - Create `ProviderPlugin` interface
   - Migrate Anthropic/OpenAI/Google to plugins
   - Enable custom provider backends

2. **Expand Error Code Vocabulary** (1 day)
   ```typescript
   const ErrorCodes = {
     // Auth errors
     UNAUTHORIZED: "UNAUTHORIZED",
     FORBIDDEN: "FORBIDDEN",
     TOKEN_EXPIRED: "TOKEN_EXPIRED",
     // Resource errors
     NOT_FOUND: "NOT_FOUND",
     ALREADY_EXISTS: "ALREADY_EXISTS",
     CONFLICT: "CONFLICT",
     // Validation errors
     INVALID_REQUEST: "INVALID_REQUEST",
     MISSING_REQUIRED_FIELD: "MISSING_REQUIRED_FIELD",
     INVALID_FIELD_VALUE: "INVALID_FIELD_VALUE",
     // Runtime errors
     AGENT_TIMEOUT: "AGENT_TIMEOUT",
     CHANNEL_NOT_LINKED: "CHANNEL_NOT_LINKED",
     MODEL_NOT_AVAILABLE: "MODEL_NOT_AVAILABLE",
     RATE_LIMIT_EXCEEDED: "RATE_LIMIT_EXCEEDED",
     // System errors
     INTERNAL_ERROR: "INTERNAL_ERROR",
     SERVICE_UNAVAILABLE: "SERVICE_UNAVAILABLE"
   };
   ```

### HIGH PRIORITY (Next Quarter)

3. **Add OpenAPI Specification** (2 days)
   - Generate from TypeBox schemas
   - Publish to `/openapi.json`
   - Enable Swagger UI at `/docs`

4. **Standardize Tool Schemas** (3 days)
   - Audit all 60+ tools for naming consistency
   - Create tool schema generator (enforce conventions)
   - Document tool authoring guidelines

5. **Implement Cursor Pagination** (1 day)
   ```typescript
   {
     "data": [...],
     "pagination": {
       "nextCursor": "cursor_xyz",
       "prevCursor": "cursor_abc",
       "hasMore": true
     }
   }
   ```

### MEDIUM PRIORITY (Future)

6. **Add HATEOAS Links** (2 days)
   - Add `_links` to all REST responses
   - Enable API discoverability
   - Reduce client hardcoding

7. **Structured CLI Errors** (1 day)
   - Add `--json` flag for structured output
   - Exit code conventions (0=success, 1=error, 2=validation)

8. **Request ID Tracking** (1 day)
   - Add `requestId` to all errors
   - Enable distributed tracing
   - Correlate logs across services

---

## For AI Coding Agents

### When Building New APIs

**DO**:
- ✅ Use TypeBox schemas for all request/response types
- ✅ Return standardized `ErrorShape` on failures
- ✅ Follow RPC naming convention (`resource.action`)
- ✅ Validate inputs with AJV (runtime type checking)
- ✅ Use discriminated unions for frame types
- ✅ Document schemas with `description` fields
- ✅ Support AbortSignal for cancellation
- ✅ Return structured results (`{ content, details }`)
- ✅ Use immutable config updates (return new object)
- ✅ Prefix plugin HTTP routes with `/api/<plugin-id>/`

**DON'T**:
- ❌ Use `Type.Union` in tool schemas (breaks Gemini)
- ❌ Throw plain `Error` objects (use `ErrorShape`)
- ❌ Mix naming conventions (stick to camelCase/kebab-case)
- ❌ Return `Promise<unknown>` (always type the response)
- ❌ Skip schema validation (always validate inputs)
- ❌ Use magic strings for error codes (use constants)
- ❌ Forget to handle AbortSignal (check `signal.aborted`)
- ❌ Use synchronous I/O in gateway methods (always async)

### Best API Examples in Codebase

**Gateway Protocol**:
- EXCELLENT: `src/gateway/protocol/schema/frames.ts` (discriminated unions)
- EXCELLENT: `src/gateway/protocol/schema/agent.ts` (comprehensive schemas)
- GOOD: `src/gateway/server-methods/agent.ts` (request handler pattern)

**Channel Plugin**:
- EXCELLENT: `extensions/bluebubbles/src/channel.ts` (complete adapter implementation)
- GOOD: `src/channels/plugins/types.plugin.ts` (plugin contract)

**Tool Definition**:
- EXCELLENT: `src/agents/tools/common.ts` (helper functions)
- GOOD: `src/agents/tools/browser-tool.ts` (complex tool with validation)

**Error Handling**:
- EXCELLENT: `src/gateway/protocol/schema/error-codes.ts` (error code constants)
- GOOD: `src/gateway/protocol/index.ts` (formatValidationErrors helper)

### Anti-Patterns to Avoid

**BAD - Inconsistent parameter names**:
```typescript
// Discord tools use both serverId and guildId
Type.Object({ serverId: Type.String() })  // File 1
Type.Object({ guildId: Type.String() })   // File 2
```

**BAD - Missing required validation**:
```typescript
// No schema validation
export async function myMethod(params: any) {
  // Directly uses params without validation
  const result = await doSomething(params.foo);
}
```

**BAD - Plain error objects**:
```typescript
throw new Error("Something failed");  // No error code!
```

**GOOD - Standardized approach**:
```typescript
import { validateMyParamsSchema, errorShape, ErrorCodes } from "../protocol/index.js";

export async function myMethod(params: unknown) {
  if (!validateMyParamsSchema(params)) {
    throw errorShape(
      ErrorCodes.INVALID_REQUEST,
      formatValidationErrors(validateMyParamsSchema.errors),
      { details: validateMyParamsSchema.errors }
    );
  }
  
  try {
    return await doSomething(params.foo);
  } catch (err) {
    throw errorShape(
      ErrorCodes.INTERNAL_ERROR,
      String(err),
      { retryable: true }
    );
  }
}
```

---

## Conclusion

OpenClaw demonstrates **EXCELLENT API design maturity** (8.5/10) with:
- Strong type safety (TypeBox + AJV)
- Consistent patterns across all surfaces
- Extensible plugin architecture
- Standardized error handling
- Production-ready quality

**Key achievement**: 2,522 TypeScript files with minimal API inconsistency

**Recommended focus**: Provider API standardization, error code expansion, OpenAPI documentation

**Risk**: LOW - APIs are stable and well-architected

**Grade**: A (Production-ready with minor improvements recommended)
