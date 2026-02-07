# OpenClaw Configuration System

## Overview

OpenClaw uses a JSON5-based configuration system with Zod schema validation. The configuration controls all aspects of the system: model selection, channel settings, agent behavior, and plugin configuration.

## Configuration Files

| File | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main user config (JSON5 format) |
| `~/.openclaw/credentials/` | Channel credentials (WhatsApp session, etc.) |
| `~/.openclaw/workspace/` | Agent workspace (AGENTS.md, SOUL.md, skills/) |
| `~/.openclaw/state/` | Runtime state (sessions, logs) |

## Config Type Hierarchy

```
src/config/
  types.ts              # Re-exports from all type files
  types.base.ts         # OpenClawConfig root type, agent config, model config
  types.channels.ts     # ChannelsConfig (aggregates all channel configs)
  types.discord.ts      # DiscordConfig
  types.googlechat.ts   # GoogleChatConfig
  types.imessage.ts     # IMessageConfig
  types.msteams.ts      # MSTeamsConfig
  types.signal.ts       # SignalConfig
  types.slack.ts        # SlackConfig
  types.telegram.ts     # TelegramConfig
  types.whatsapp.ts     # WhatsAppConfig
```

## Root Config Shape (OpenClawConfig)

```typescript
type OpenClawConfig = {
  // Core
  model?: string;              // Default model (e.g., "claude-3.5-sonnet")
  agentModel?: string;         // Agent-specific model override
  maxTokens?: number;          // Max output tokens
  thinking?: boolean | object; // Extended thinking mode

  // Agent
  agent?: AgentConfig;         // Agent behavior settings
  agents?: Record<string, AgentConfig>; // Named agent configs

  // Channels
  channels?: ChannelsConfig;   // Per-channel configuration

  // Routing
  routing?: RoutingConfig;     // Message routing rules
  allowFrom?: string[];        // Global allowlist

  // Plugins
  plugins?: PluginConfig[];    // Plugin references
  pluginConfig?: Record<string, unknown>; // Per-plugin settings

  // Features
  cron?: CronConfig;           // Scheduled tasks
  memory?: MemoryConfig;       // Vector memory settings
  browser?: BrowserConfig;     // Browser control settings

  // Provider auth
  profiles?: AuthProfiles;     // API key/token profiles
};
```

## Channel Configuration Pattern

Each channel follows a consistent pattern:

```typescript
type ChannelConfig = {
  // Account(s) - either single or multi-account
  token?: string;              // API token
  accounts?: Record<string, AccountConfig>;

  // Access control
  dm?: DmConfig;               // Direct message policy
  groupPolicy?: GroupPolicy;   // Group chat policy

  // Behavior
  enabled?: boolean;
  streaming?: StreamingConfig;
  markdown?: MarkdownConfig;

  // Channel-specific settings
  // ...
};

type DmConfig = {
  allowFrom?: string[];        // Allowed sender IDs
  policy?: DmPolicy;           // "allowlist" | "open" | "disabled"
};

type GroupPolicy = {
  requireMention?: boolean;    // Must @mention bot in groups
  tools?: GroupToolPolicyConfig;
};
```

## Zod Schema Validation

Config validation uses Zod schemas:

```
src/config/
  zod-schema.ts               # Root OpenClawSchema
  zod-schema.core.ts          # Core schemas (DmConfig, GroupPolicy, etc.)
  zod-schema.agent-runtime.ts # Agent config schemas
  zod-schema.providers-core.ts # Provider channel schemas
  zod-schema.providers-whatsapp.ts # WhatsApp-specific schemas
```

### Using Zod Schemas

```typescript
import { OpenClawSchema } from "../config/zod-schema.js";

// Validate config
const result = OpenClawSchema.safeParse(rawConfig);
if (!result.success) {
  console.error("Invalid config:", result.error.issues);
}
```

### Adding New Config Section

1. Define TypeScript types in `src/config/types.<section>.ts`
2. Define Zod schema in `src/config/zod-schema.<section>.ts`
3. Add to root `OpenClawConfig` type
4. Add to root `OpenClawSchema`
5. Export from `src/config/config.ts`
6. Add validation in `src/config/validation.ts`

## Config IO

```typescript
import { loadConfig, writeConfigFile, createConfigIO } from "../config/config.js";

// Load config
const config = await loadConfig();

// Write config
await writeConfigFile(configPath, newConfig);

// ConfigIO pattern (for DI)
const io = createConfigIO();
const config = await io.load();
await io.save(updatedConfig);
```

## Config Paths

```typescript
import { resolveConfigDir, resolveWorkspaceDir, resolveStateDir } from "../config/paths.js";

const configDir = resolveConfigDir();      // ~/.openclaw/
const workspaceDir = resolveWorkspaceDir(); // ~/.openclaw/workspace/
const stateDir = resolveStateDir();         // ~/.openclaw/state/
```

## Runtime Overrides

Environment variables can override config values:

```typescript
import { applyRuntimeOverrides } from "../config/runtime-overrides.js";

// Environment variables like OPENCLAW_MODEL, OPENCLAW_MAX_TOKENS, etc.
const config = applyRuntimeOverrides(baseConfig);
```

## Hot Reload

The gateway supports config hot-reload:

```typescript
// src/gateway/config-reload.ts
// Watches config file for changes and applies them without restart
// Reloads: model settings, channel configs, routing rules
// Does NOT reload: plugins, gateway port, credentials
```

## Plugin Configuration

Plugins define their config schema and receive their section:

```typescript
// In plugin definition
const plugin = {
  id: "my-plugin",
  configSchema: {
    safeParse: (value) => MyPluginSchema.safeParse(value),
    uiHints: {
      apiKey: { label: "API Key", sensitive: true },
    },
  },
};

// In openclaw.json
{
  "plugins": ["@openclaw/my-plugin"],
  "pluginConfig": {
    "my-plugin": {
      "apiKey": "sk-...",
      "enabled": true
    }
  }
}

// Access in plugin
api.pluginConfig; // { apiKey: "sk-...", enabled: true }
```

## Config for Verticals

Industry verticals add their config under `pluginConfig`:

```json5
{
  "model": "claude-3.5-sonnet",
  "channels": {
    "whatsapp": { "dm": { "allowFrom": ["*"] } }
  },
  "plugins": ["@openclaw/restaurant"],
  "pluginConfig": {
    "openclaw-restaurant": {
      "restaurantName": "Joe's Diner",
      "pos": {
        "provider": "toast",
        "apiKey": "toast_key_..."
      },
      "reservations": {
        "provider": "builtin",
        "maxPartySize": 12
      }
    }
  }
}
```
