# OpenClaw Codebase - Quick Reference Guide

**For AI Coding Agents** | **Last Updated**: February 6, 2026

---

## What Is OpenClaw?

A **personal AI assistant gateway** that:
- Routes messages from 20+ messaging platforms (Discord, Slack, Telegram, etc.)
- Connects to multiple AI model providers (Claude, GPT-4, Gemini, etc.)
- Runs agent tools (shell commands, file operations, web browsing)
- Manages conversation sessions and memory
- Supports native companion apps (macOS, iOS, Android)

---

## Architecture in 60 Seconds

```
User sends message on Discord
           ↓
      Discord Plugin
           ↓
    WebSocket Gateway (port 18789)
           ↓
      Pi Agent Process
           ↓
    Model Provider (Anthropic, OpenAI, etc.)
           ↓
    Agent Tools (bash, read, write, canvas, etc.)
           ↓
    Response back through Gateway
           ↓
    Discord Plugin sends to user
```

**Key Insight**: All I/O goes through the Gateway. This is intentional—it enables session routing, reliable delivery, and multi-agent support.

---

## Where Is Everything?

### Core Executable
- `/src/entry.ts` - CLI entrypoint
- `/src/cli/` - Command infrastructure
- `/src/commands/` - CLI commands (agent, gateway, send, doctor, etc.)

### Message Routing
- `/src/gateway/` - WebSocket server (the "hub")
- `/src/channels/` - Shared channel abstractions
- `/src/discord/`, `/src/slack/`, `/src/telegram/`, etc. - Channel implementations
- `/extensions/` - Additional channel plugins (30 of them)

### AI Execution
- `/src/agents/` - Agent runtime (400+ files)
- `/src/agents/pi-embedded-runner.ts` - Execute Pi agent
- `/src/agents/pi-embedded-subscribe.ts` - Stream responses
- `/src/agents/bash-tools.ts` - Shell execution
- `/src/agents/pi-tools.ts` - All agent tools
- `/src/agents/system-prompt.ts` - Dynamic system prompt

### Configuration
- `/src/config/` - Load `~/.openclaw/openclaw.json`
- `/src/infra/` - System infrastructure
- `/src/sessions/` - Session state management

### Everything Else
- `/src/media/` - Image/video processing
- `/src/providers/` - Model provider integrations
- `/src/browser/` - Playwright web automation
- `/src/tui/`, `/src/web/` - UI interfaces
- `/apps/` - Native companion apps (Swift + Kotlin)

---

## Most Important Files

### For Understanding Architecture
1. `/src/entry.ts` - How CLI starts up
2. `/src/gateway/server.ts` - Gateway initialization
3. `/src/agents/pi-embedded-runner.ts` - Agent execution
4. `/src/channels/dock.ts` - Channel metadata system
5. `/src/plugins/runtime.ts` - Extension loading

### For Making Changes
1. `/src/agents/pi-tools.ts` - Add agent tools here
2. `/src/agents/system-prompt.ts` - Modify system prompt
3. `/extensions/*/index.ts` - Add channel features
4. `/src/config/config.ts` - Config validation
5. `/src/commands/` - Add CLI commands

### For Testing
1. `/vitest.config.ts` - Unit test config
2. `/vitest.e2e.config.ts` - E2E test config
3. `/test/setup.ts` - Test setup

---

## Key Design Patterns

### 1. Gateway as Control Plane (Not Direct Channel ↔ Agent)
**Why**: Single routing point, session management, reliable delivery
**File**: `/src/gateway/server.ts`

### 2. Plugin Architecture (Channels Are Loadable)
**Why**: Support 20+ platforms without recompiling
**Pattern**: Export `ChannelPlugin` interface from `extensions/*/index.ts`
**How It Works**: `/src/plugins/runtime.ts` auto-discovers and loads extensions

### 3. Separate Process for Agents (Not In-Gateway)
**Why**: Isolate crashes, enable sandboxing, allow parallel execution
**How**: Gateway spawns child process, communicates via RPC
**File**: `/src/gateway/server-bridge.ts`

### 4. Multi-Provider Model Fallback
**Why**: Resilience (if OpenAI down, use Anthropic), cost optimization
**Files**: `/src/agents/model-fallback.ts`, `/src/agents/model-selection.ts`

### 5. Dependency Injection Pattern
**Why**: Easy testing, swappable implementations
**How**: Use `createDefaultDeps()` from `/src/cli/deps.ts`

---

## Common Tasks

### Add a New Channel
1. Create `extensions/my-channel/` directory
2. Write `extensions/my-channel/package.json` with dependency
3. Write `extensions/my-channel/index.ts` exporting `ChannelPlugin`
4. Implement message receive/send handlers
5. Update `~/.openclaw/openclaw.json` to enable channel

### Add a New Agent Tool
1. Edit `/src/agents/pi-tools.ts`
2. Add tool definition (name, description, input schema, execute function)
3. Update `/src/agents/pi-tools.policy.ts` if approval needed
4. Test via `pnpm test:watch`

### Add a New Model Provider
1. Update `/src/agents/models-config.providers.ts`
2. Add auth/token resolution
3. Add model enumeration
4. Update `/src/agents/model-fallback.ts` if fallback logic needed

### Debug an Issue
1. Check logs: `pnpm openclaw doctor`
2. Enable verbose mode: `OPENCLAW_LOG_LEVEL=debug`
3. Check session state: `~/.openclaw/workspace/sessions.json`
4. Run in watch mode: `pnpm gateway:watch` + `pnpm test:watch`

---

## Testing

```bash
pnpm test              # Unit tests (fast)
pnpm test:watch       # Watch mode
pnpm test:e2e         # Integration tests
pnpm test:live        # Real API tests (requires keys)
pnpm test:coverage    # Coverage report (target: 70%)
```

**Coverage Target**: 70% overall
- Excluded: CLI, gateway surfaces, interactive UIs, channel integrations (integration tested)

---

## Development Workflow

```bash
# Install dependencies
pnpm install

# Start gateway with auto-reload
pnpm gateway:watch

# In another terminal, start agent in RPC mode
pnpm openclaw:rpc

# Run tests in watch mode
pnpm test:watch

# Build for production
pnpm build

# Check code quality
pnpm check  # = lint + format check
```

---

## Configuration

### `~/.openclaw/openclaw.json` (User Config)
```json
{
  "agents": {
    "default": {
      "name": "Claude",
      "models": ["claude-3-5-sonnet"],
      "provider": "anthropic"
    }
  },
  "channels": {
    "discord": { "enabled": true },
    "telegram": { "enabled": true }
  }
}
```

### Environment Variables
- `OPENCLAW_LOG_LEVEL` - Logging level (debug, info, warn, error)
- `OPENCLAW_LIVE_TEST` - Enable live API tests
- `NODE_OPTIONS` - Node.js options (auto-handled by entry.ts)
- `NO_COLOR` - Disable colored output

---

## Important Files Cheat Sheet

| File | Purpose | When to Edit |
|------|---------|--------------|
| `/src/agents/pi-tools.ts` | Agent tool definitions | Adding new capabilities |
| `/src/agents/system-prompt.ts` | AI system prompt | Changing agent behavior |
| `/src/gateway/server.ts` | Gateway server | Changing gateway behavior |
| `/src/channels/dock.ts` | Channel metadata | Channel capabilities, mention patterns |
| `/src/config/config.ts` | Config loading | User configuration schema |
| `/extensions/*/index.ts` | Channel implementation | Adding/modifying channels |
| `/src/agents/model-selection.ts` | Model choice logic | Model selection strategy |
| `/src/agents/bash-tools.ts` | Shell execution | Shell command handling |
| `/tsconfig.json` | TypeScript config | TypeScript strictness |
| `/vitest.config.ts` | Test configuration | Test setup |

---

## Module Dependency Quick Map

```
┌─────────────────────────────────────┐
│   CLI / Commands                    │
├─────────────────────────────────────┤
│   Config / Infrastructure / Logging │
├─────────────────────────────────────┤
│   Gateway (WebSocket hub)           │
├─────────────────────────────────────┤
│   Channels + Agent Runtime          │
├─────────────────────────────────────┤
│   Models / Providers / Tools        │
├─────────────────────────────────────┤
│   Media / Sessions / Memory         │
├─────────────────────────────────────┤
│   Utilities / Types / Shared        │
└─────────────────────────────────────┘
```

**Rule**: Outer layers depend on inner layers, never reverse.

---

## Performance Characteristics

- **Gateway throughput**: 100+ concurrent connections
- **Message latency**: <10ms routing (dominated by model inference)
- **Memory per session**: 1-5 MB (depends on history size)
- **Agent process memory**: 500 MB - 2 GB

**Bottleneck**: Model inference (not gateway)

---

## Glossary (Use These Terms Consistently)

- **Agent** - Pi agent entity that processes messages
- **Channel** - Messaging platform (Discord, Telegram, etc.)
- **Extension** - Loadable plugin (channel, tool, auth, etc.)
- **Gateway** - Central WebSocket control plane
- **Session** - Conversation state for one user+channel
- **Tool** - Action agent can execute
- **Model** - LLM (Claude, GPT-4, Gemini)
- **Provider** - API provider (Anthropic, OpenAI, Google)
- **Sandbox** - Isolated code execution environment
- **Dock** - Channel metadata (capabilities, rules)
- **RPC** - Remote Procedure Call (gateway ↔ agent)
- **Subagent** - Agent spawned by another agent
- **Skill** - User-defined agent capability
- **Canvas** - Real-time drawing/SVG surface

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Gateway won't start | Check port 18789 available; see `pnpm openclaw doctor` |
| Agent crashes | Check system prompt; look at error logs |
| Model not found | Verify `~/.openclaw/openclaw.json` config |
| Tool permission denied | Check `/src/agents/pi-tools.policy.ts` |
| Extension not loading | Verify `extensions/*/openclaw.plugin.json` manifest |
| Tests failing | Run `pnpm format:fix && pnpm lint:fix` first |

---

## For More Details

- **Full architectural analysis**: See `STRUCTURE_ANALYSIS.md` (1,400+ lines)
- **JSON reference map**: See `STRUCTURE_MAP.json` (programmatic access)
- **Build commands**: See `/CLAUDE.md` (project guidelines)
- **Protocol details**: See `/src/gateway/protocol/` (wire protocol)

---

**Last verified**: February 6, 2026  
**Ready for AI agent steering context**: Yes
