# Gateway Protocol Reference

The OpenClaw Gateway is a WebSocket server (default port 18789) that serves as the central control plane.

## 1. Frame Types (JSON over WS)

### Request
```json
{ "id": 1, "method": "sessions.list", "params": {} }
```
### Response
```json
{ "id": 1, "result": [...] }
```
### Event (Server -> Client)
```json
{ "event": "agent_event", "data": { "type": "thinking", ... } }
```

## 2. Essential RPC Methods

### Sessions
- `sessions.list`: List active sessions.
- `sessions.resolve`: Resolve session by key.
- `sessions.patch`: Update metadata.
- `sessions.compact`: Force history compaction.

### Messaging & Agent
- `send`: Send raw message to target.
- `agent`: Invoke agent with message (starts a run).
- `agent.wait`: Wait for current run to finish.
- `chat.abort`: Interrupt the running agent.

### System
- `config.get` / `config.patch`: Manage configuration.
- `channels.status`: Check connectivity.
- `models.list`: List available LLMs.
- `nodes.list`: List connected companion apps.

## 3. Server Events
- `agent_event`: Streaming updates (thought, tool_call, delta).
- `chat_event`: Incoming/Outgoing message notifications.
- `snapshot`: Full state dump on connect.
- `tick`: 30s heartbeat.

## 4. Authorization Roles
- `operator.admin`: Full control (config, skills, destructive actions).
- `operator.write`: Can send messages and invoke agents.
- `operator.read`: Can view logs, status, and sessions.
- `operator.approvals`: Can approve tool executions.

## 5. Schema Locations
- **Types:** `src/gateway/protocol/schema/types.ts`
- **Validators:** `src/gateway/protocol/index.ts`
- **Handlers:** `src/gateway/server-methods/`
