# OpenClaw Domain Model & Business Rules

## Core Entities

### Agent
- **Type**: Aggregate Root
- **Purpose**: AI assistant persona with its own model, workspace, tools, and sessions
- **Key Attributes**: id, model ref, thinking level, tool profile, workspace, sandbox config
- **Business Rules**:
  - Default agent ID: `"default"` (always exists)
  - Model selection cascade: agent config -> global config -> fallback -> default
  - Tool policies use deny-wins semantics (if any rule denies, tool is denied)
  - Thinking levels: off, minimal, low, medium, high, xhigh

### Session
- **Type**: Entity (managed lifecycle)
- **Purpose**: Conversation context identified by a deterministic key
- **Key**: `agent:{agentId}:{scope}` (hierarchical, deterministic)
- **Business Rules**:
  - Session key MUST be deterministic (same inputs = same key) - this is CRITICAL
  - Per-peer vs per-agent scope configurable per channel
  - Auto-reset on configurable idle timeout or daily boundary
  - Compaction triggers when tokens exceed `contextWindowTokens * 1.2`

### Channel
- **Type**: Integration adapter
- **Purpose**: Messaging platform connection (WhatsApp, Telegram, etc.)
- **Key Attributes**: id, capabilities, DM policy, group policy, allowlist

### Model Provider
- **Type**: Service adapter
- **Purpose**: LLM API integration with failover
- **Business Rules**:
  - Fallback chain: primary model -> fallbackModels[0] -> fallbackModels[1] -> ...
  - Auth profile cooldown: failed profiles get 60-second cooldown
  - Failover errors must propagate (never swallow FailoverErrors)

## Authorization Matrix

Every WebSocket RPC method requires appropriate role and scope:

| Scope | Methods |
|-------|---------|
| `operator.admin` | config.*, wizard.*, update.*, channels.logout, skills.install, sessions.patch/reset/delete/compact |
| `operator.read` | health, logs.tail, channels.status, models.list, agents.list, sessions.list/preview |
| `operator.write` | send, agent, agent.wait, wake, talk.mode, node.invoke, chat.send/abort |
| `operator.approvals` | exec.approval.request, exec.approval.resolve |
| `operator.pairing` | node.pair.*, device.pair.*, device.token.*, node.rename |

IMPORTANT: `operator.admin` scope bypasses all other checks. Admin tokens are root credentials.

## Message Processing Pipeline

```
received -> security_gate -> command_detection -> build_envelope -> resolve_session -> dispatch_to_agent
                                                                                           |
                                                                      stream_response <- agent_running
                                                                           |
                                                                      deliver_reply
```

### Security Gates
1. DM policy check (open/allowlist/pairing/disabled)
2. Group mention requirement check
3. Command gating (admin-only commands)
4. Send policy evaluation

### Inbound Envelope Format
```
[{channel} {from} {timestamp}] {body}
```
Example: `[WhatsApp Alice 2026-01-15 09:30 EST] Hello, can I make a reservation?`

## Critical Business Rules (NEVER Violate)

1. **Session key determinism**: `buildAgentPeerSessionKey()` must be a pure function of inputs
2. **Auth scope enforcement**: `authorizeGatewayMethod()` must check scopes before handler execution
3. **Model allowlist**: `buildAllowedModelSet()` must restrict in-chat model switches
4. **Sandbox isolation**: `assertSandboxPath()` must reject paths outside sandbox root
5. **SSRF prevention**: `resolvePinnedHostname()` must be used for all user-provided URL fetches
6. **Timing-safe auth**: `safeEqual()` must be used for all credential comparison
7. **Media TTL cleanup**: Stored media must have bounded TTL and automatic cleanup

## Compaction Formula

**Trigger**: When `estimatedTokens >= contextWindowTokens * SAFETY_MARGIN (1.2)`

**Constants**:
- `BASE_CHUNK_RATIO` = 0.4 (split at 40% of history)
- `MIN_CHUNK_RATIO` = 0.15 (minimum chunk size)
- `DEFAULT_CONTEXT_TOKENS` = 200,000

**Process**: Split conversation into N parts, summarize each via LLM, merge summaries.

## Chat Delta Throttling

Agent streaming deltas emitted to WebSocket clients at most every **150ms** per run.

## Key Files for Domain Logic

| Purpose | File |
|---------|------|
| Session key construction | `src/routing/session-key.ts` |
| Agent scope/config | `src/agents/agent-scope.ts` |
| Agent identity | `src/agents/identity.ts` |
| Model selection | `src/agents/model-selection.ts` |
| Model fallback | `src/agents/model-fallback.ts` |
| Context compaction | `src/agents/compaction.ts` |
| Auto-reply dispatch | `src/auto-reply/dispatch.ts` |
| Inbound envelope | `src/auto-reply/envelope.ts` |
| Gateway auth | `src/gateway/auth.ts` |
| Gateway RPC handlers | `src/gateway/server-methods.ts` |
| Process lanes | `src/process/lanes.ts` |
| Media store | `src/media/store.ts` |
| Cron service | `src/cron/service.ts` |
| Hook types | `src/hooks/types.ts` |

## Bounded Contexts

| Context | Entities | Language |
|---------|----------|----------|
| Agent Runtime | Agent, Session, Tool, Model, Compaction | "agent", "session key", "model ref", "thinking level" |
| Channel Abstraction | Channel, Plugin, DmPolicy, GroupPolicy | "channel", "peer", "mention gating", "allowlist" |
| Gateway Control Plane | Gateway, Node, WebSocket Client, RPC | "connect", "broadcast", "scope", "role", "tick" |
| Scheduling | CronJob, Hook, HeartbeatRunner | "cron expression", "schedule", "isolated run" |
| Configuration | Config, Schema, Migration, Validation | "config", "schema", "hot reload", "doctor" |
| Media Pipeline | MediaStore, Understanding, LinkUnderstanding | "MIME", "transcription", "vision", "media buffer" |
| Security | ExecApproval, Sandbox, Audit, SSRF | "sandbox", "elevated", "path validation" |

## Ubiquitous Language

| Term | Definition |
|------|-----------|
| Agent | AI assistant persona with model, workspace, tools, sessions |
| Session Key | Hierarchical identifier: `agent:{id}:{scope}` |
| Channel | Messaging platform integration |
| Gateway | Central WebSocket/HTTP server and control plane |
| Model Ref | Provider/model identifier like `anthropic/claude-opus-4-5` |
| Thinking Level | Reasoning intensity: off, minimal, low, medium, high, xhigh |
| Fallback | Next model to try when primary fails |
| Skill | Modular agent capability extension with SKILL.md |
| Hook | Event-reactive automation module with HOOK.md |
| Tool Profile | Preset tool allowlist (minimal, coding, messaging, full) |
| Elevated Mode | Temporary exec permission expansion via /elevated |
| Binding | Route rule mapping channel+peer to specific agent |
| DM Scope | How DM sessions are partitioned (per-peer or per-agent) |
| Envelope | Formatted message header prepended to agent input |
| Compaction | Summarizing old history to save context tokens |
| Block Streaming | Delivering responses in message-sized blocks |
| Node | Connected companion app or remote device |
| Control UI | Browser-based gateway management interface |
