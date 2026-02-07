# Gateway Protocol Guide

## Overview

The OpenClaw Gateway is a WebSocket server (default port 18789) that serves as the central control plane. All clients (CLI, companion apps, channel bridges) communicate through it.

## Connection Flow

```
Client                          Gateway
  |                                |
  |--- WebSocket Connect --------->|
  |                                |
  |<-- HelloOk (protocol ver) ----|
  |                                |
  |--- connect { token, ... } --->|
  |<-- connect response ----------|
  |                                |
  |--- Snapshot Event ------------>|  (Full state snapshot)
  |                                |
  |--- RPC Request/Response ------>|  (Ongoing communication)
  |<-- Events ---------------------|
```

## Frame Types

All communication uses JSON frames over WebSocket:

### RequestFrame

```typescript
type RequestFrame = {
  id: number;          // Unique request ID
  method: string;      // RPC method name
  params?: unknown;    // Method parameters
};
```

### ResponseFrame

```typescript
type ResponseFrame = {
  id: number;          // Matches request ID
  result?: unknown;    // Success result
  error?: ErrorShape;  // Error (if failed)
};
```

### EventFrame

```typescript
type EventFrame = {
  event: string;       // Event name
  data?: unknown;      // Event payload
};
```

## Key RPC Methods

### Session Management

| Method | Description |
|--------|-------------|
| `sessions.list` | List all active sessions |
| `sessions.preview` | Preview session messages |
| `sessions.resolve` | Resolve session by key |
| `sessions.patch` | Update session metadata |
| `sessions.reset` | Reset session state |
| `sessions.delete` | Delete a session |
| `sessions.compact` | Compact session history |

### Message Sending

| Method | Description |
|--------|-------------|
| `send` | Send message to target |
| `poll` | Poll for new messages |
| `chat.send` | Send in chat context |
| `chat.history` | Get chat history |
| `chat.abort` | Abort running agent |
| `chat.inject` | Inject message into session |

### Agent Control

| Method | Description |
|--------|-------------|
| `agent` | Invoke agent with message |
| `agent.identity` | Get/set agent identity |
| `agent.wait` | Wait for agent completion |
| `agents.list` | List available agents |

### Configuration

| Method | Description |
|--------|-------------|
| `config.get` | Get config value |
| `config.set` | Set config value |
| `config.apply` | Apply config changes |
| `config.patch` | Patch config object |
| `config.schema` | Get config schema |

### Channels

| Method | Description |
|--------|-------------|
| `channels.status` | Get channel status |
| `channels.logout` | Logout from channel |

### Skills & Models

| Method | Description |
|--------|-------------|
| `models.list` | List available models |
| `skills.status` | Get skills status |
| `skills.install` | Install a skill |
| `skills.update` | Update skills |

### Cron Jobs

| Method | Description |
|--------|-------------|
| `cron.list` | List cron jobs |
| `cron.add` | Add cron job |
| `cron.update` | Update cron job |
| `cron.remove` | Remove cron job |
| `cron.run` | Manually run cron job |
| `cron.runs` | Get cron run history |

### Node Management

| Method | Description |
|--------|-------------|
| `node.list` | List connected nodes |
| `node.describe` | Describe a node |
| `node.invoke` | Invoke method on node |
| `node.pair.request` | Request node pairing |
| `node.pair.approve` | Approve pairing request |

### Device Management

| Method | Description |
|--------|-------------|
| `device.pair.list` | List paired devices |
| `device.pair.approve` | Approve device pairing |
| `device.token.rotate` | Rotate device token |

### Setup Wizard

| Method | Description |
|--------|-------------|
| `wizard.start` | Start setup wizard |
| `wizard.next` | Advance wizard step |
| `wizard.cancel` | Cancel wizard |
| `wizard.status` | Get wizard status |

### Logs

| Method | Description |
|--------|-------------|
| `logs.tail` | Tail gateway logs |

## Events (Server -> Client)

| Event | Description |
|-------|-------------|
| `agent_event` | Agent activity (thinking, tool use, response) |
| `chat_event` | Chat message events |
| `tick` | Periodic heartbeat |
| `shutdown` | Gateway shutting down |
| `snapshot` | Full state snapshot |
| `state_version` | State version update |

## Error Codes

```typescript
const ErrorCodes = {
  ParseError: -32700,
  InvalidRequest: -32600,
  MethodNotFound: -32601,
  InvalidParams: -32602,
  InternalError: -32603,
  // Custom codes
  Unauthorized: -32000,
  NotFound: -32001,
  Conflict: -32002,
  // ...
};
```

## Protocol Schema Location

All schemas defined in `src/gateway/protocol/schema/`:

```
schema/
  agent.ts            # Agent-related schemas
  agents-models-skills.ts  # Listing schemas
  channels.ts         # Channel schemas
  config.ts           # Config schemas
  cron.ts             # Cron job schemas
  devices.ts          # Device management schemas
  error-codes.ts      # Error code definitions
  exec-approvals.ts   # Execution approval schemas
  frames.ts           # Wire frame schemas
  logs-chat.ts        # Log and chat schemas
  nodes.ts            # Node management schemas
  primitives.ts       # Primitive type schemas
  protocol-schemas.ts # Protocol schema registry
  sessions.ts         # Session schemas
  snapshot.ts         # Snapshot schema
  types.ts            # Shared types
  wizard.ts           # Setup wizard schemas
```

## Schema Validation

All params are validated at the gateway boundary using AJV-compiled validators:

```typescript
import { validateSendParams } from "../gateway/protocol/index.js";

// In a gateway method handler:
if (!validateSendParams(params)) {
  return errorShape(ErrorCodes.InvalidParams, formatValidationErrors(validateSendParams.errors));
}
```

## HTTP Endpoints

The gateway also serves HTTP endpoints:

- **Control UI**: Web-based admin interface
- **Webhooks**: Channel-specific webhook receivers
- **OpenAI-compatible API**: `POST /v1/chat/completions` (and Open Responses API)
- **Plugin routes**: Custom HTTP routes registered by plugins

## Adding a New Gateway Method

1. Define schema in `src/gateway/protocol/schema/<domain>.ts`
2. Export from `src/gateway/protocol/schema.ts`
3. Create validator in `src/gateway/protocol/index.ts`
4. Implement handler in `src/gateway/server-methods/<domain>.ts`
5. Register in `src/gateway/server-methods-list.ts`
6. Regenerate protocol: `pnpm protocol:gen` (and `pnpm protocol:gen:swift` for companion apps)
