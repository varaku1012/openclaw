# Channel Development Guide

## Overview

Channels connect OpenClaw to messaging platforms. Each channel implements the `ChannelPlugin` interface and is registered via the Plugin SDK.

## Channel Architecture

```
User on Platform  -->  Platform API  -->  Channel Plugin  -->  Gateway  -->  Agent
                                                                 |
                                                                 v
                                                          Session Manager
```

## ChannelPlugin Interface

The full interface (`src/channels/plugins/types.plugin.ts`):

```typescript
type ChannelPlugin<ResolvedAccount = any> = {
  // Required
  id: ChannelId;                         // Unique identifier: "mychannel"
  meta: ChannelMeta;                     // { name, icon, color }
  capabilities: ChannelCapabilities;     // What the channel supports
  config: ChannelConfigAdapter<RA>;      // Account resolution from config

  // Optional adapters
  setup?: ChannelSetupAdapter;           // First-time setup wizard
  pairing?: ChannelPairingAdapter;       // QR code or token pairing
  security?: ChannelSecurityAdapter;     // Access control
  groups?: ChannelGroupAdapter;          // Group chat handling
  mentions?: ChannelMentionAdapter;      // @mention processing
  outbound?: ChannelOutboundAdapter;     // Sending messages
  gateway?: ChannelGatewayAdapter;       // Gateway boot/shutdown
  streaming?: ChannelStreamingAdapter;   // Block streaming
  threading?: ChannelThreadingAdapter;   // Thread support
  messaging?: ChannelMessagingAdapter;   // Inbound message processing
  directory?: ChannelDirectoryAdapter;   // Contact/group directory
  resolver?: ChannelResolverAdapter;     // ID resolution
  actions?: ChannelMessageActionAdapter; // Message actions (react, pin)
  heartbeat?: ChannelHeartbeatAdapter;   // Health monitoring
  agentPrompt?: ChannelAgentPromptAdapter; // Channel-specific prompt additions
  commands?: ChannelCommandAdapter;      // Channel-specific commands
  elevated?: ChannelElevatedAdapter;     // Elevated permissions
  agentTools?: ChannelAgentToolFactory;  // Channel-specific tools

  // Config
  configSchema?: ChannelConfigSchema;    // JSON schema + UI hints
  defaults?: { queue?: { debounceMs?: number } };
  reload?: { configPrefixes: string[] };
  onboarding?: ChannelOnboardingAdapter; // CLI onboarding wizard
};
```

## Channel Capabilities

```typescript
type ChannelCapabilities = {
  chatTypes: Array<"direct" | "group" | "channel" | "thread">;
  nativeCommands?: boolean;   // Platform supports slash commands
  blockStreaming?: boolean;   // Can stream message blocks
  reactions?: boolean;        // Supports message reactions
  edits?: boolean;            // Supports message editing
  threads?: boolean;          // Supports threaded replies
  attachments?: boolean;      // Supports file attachments
  polls?: boolean;            // Supports native polls
};
```

## Creating a New Channel

### Step 1: Extension Package

```
extensions/<channel>/
  package.json
  openclaw.plugin.json
  index.ts
  src/
    plugin.ts           # ChannelPlugin implementation
    accounts.ts         # Account resolution
    messaging.ts        # Message handling
    outbound.ts         # Sending messages
    config-schema.ts    # Configuration schema
```

### Step 2: Config Adapter (Required)

```typescript
import type { ChannelConfigAdapter } from "openclaw/plugin-sdk";

type ResolvedAccount = {
  id: string;
  token: string;
  enabled: boolean;
};

export const configAdapter: ChannelConfigAdapter<ResolvedAccount> = {
  resolveAccount(config, accountId) {
    const section = config.channels?.mychannel;
    if (!section) return null;
    return {
      id: accountId ?? "default",
      token: section.token,
      enabled: section.enabled !== false,
    };
  },
  listAccountIds(config) {
    return config.channels?.mychannel ? ["default"] : [];
  },
};
```

### Step 3: Gateway Adapter (Connects to platform)

```typescript
import type { ChannelGatewayAdapter } from "openclaw/plugin-sdk";

export const gatewayAdapter: ChannelGatewayAdapter<ResolvedAccount> = {
  async boot(account, ctx) {
    // ctx.config, ctx.logger, ctx.gateway
    // Initialize platform SDK, set up event listeners
    const client = new PlatformSDK(account.token);

    client.on("message", async (msg) => {
      // Route inbound message to gateway
      await ctx.gateway.handleInbound({
        channelId: "mychannel",
        accountId: account.id,
        from: msg.sender,
        text: msg.text,
        conversationId: msg.chatId,
      });
    });

    await client.connect();
    return { client }; // Return cleanup handle
  },

  async shutdown(handle) {
    await handle.client.disconnect();
  },
};
```

### Step 4: Outbound Adapter (Sends messages)

```typescript
import type { ChannelOutboundAdapter } from "openclaw/plugin-sdk";

export const outboundAdapter: ChannelOutboundAdapter = {
  async send(target, message, ctx) {
    // target.channelId, target.accountId, target.conversationId
    // message.text, message.media, message.replyTo
    const client = ctx.getHandle()?.client;
    if (!client) throw new Error("Not connected");

    await client.sendMessage(target.conversationId, {
      text: message.text,
    });
  },
};
```

### Step 5: Register the Plugin

```typescript
// extensions/<channel>/index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { myChannelPlugin } from "./src/plugin.js";

export default function register(api: OpenClawPluginApi) {
  api.registerChannel({ plugin: myChannelPlugin });
}
```

## Configuration Schema

Define the channel's configuration shape:

```typescript
import { buildChannelConfigSchema, DmConfigSchema } from "openclaw/plugin-sdk";
import { z } from "zod";

export const MyChannelConfigSchema = z.object({
  token: z.string().describe("Platform API token"),
  enabled: z.boolean().default(true),
  dm: DmConfigSchema.optional(),
});

// For the plugin:
configSchema: buildChannelConfigSchema({
  schema: MyChannelConfigSchema,
  uiHints: {
    token: { label: "API Token", sensitive: true },
  },
});
```

## User config (openclaw.json)

```json5
{
  "channels": {
    "mychannel": {
      "token": "platform-api-token",
      "enabled": true,
      "dm": {
        "allowFrom": ["user1", "user2"],
        "policy": "allowlist"
      }
    }
  }
}
```

## Channel Dock Registration

For shared code paths, also register a dock in `src/channels/dock.ts`:

```typescript
const dock: ChannelDock = {
  id: "mychannel",
  capabilities: {
    chatTypes: ["direct", "group"],
  },
  outbound: { textChunkLimit: 4000 },
};
```

## Testing Channels

- Unit test each adapter independently using dependency injection
- Mock the platform SDK/API client
- Test config resolution with various config shapes
- Test message routing (inbound -> gateway -> agent -> outbound)
- E2E test with the gateway using `vitest` and `ws` library

## Existing Channel Examples

Study these implementations for patterns:
- **Telegram** (`src/telegram/`, `extensions/telegram/`) - Grammy bot, comprehensive example
- **Discord** (`src/discord/`, `extensions/discord/`) - @buape/carbon library
- **Slack** (`src/slack/`, `extensions/slack/`) - @slack/bolt framework
- **WhatsApp** (`src/whatsapp/`, `extensions/whatsapp/`) - Baileys, most complex (QR pairing)
- **LINE** (`src/line/`, `extensions/line/`) - LINE Bot SDK, good flex message patterns
