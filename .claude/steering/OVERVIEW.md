# OpenClaw Steering Context - System Overview

> This steering context is designed for AI Coding Agents working on OpenClaw.
> Read this file first before any development task.

## What is OpenClaw?

OpenClaw is a **personal AI assistant engine** with a gateway-centric architecture. It connects AI agents (powered by LLMs like Anthropic Claude, OpenAI GPT, etc.) to messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, Line, and more) through a central WebSocket gateway.

**Version:** 2026.2.x | **License:** MIT | **Runtime:** Node 22+ (ESM) | **Language:** TypeScript (strict)

## High-Level Architecture

```
Messaging Channels (WhatsApp/Telegram/Slack/Discord/Signal/iMessage/Teams/Line/Matrix/etc.)
                |
                v
+---------------------------------------+
|              Gateway                   |
|     (WebSocket control plane)          |
|        ws://127.0.0.1:18789            |
+---------------+-----------------------+
                |
    +-----------+-----------+
    |           |           |
    v           v           v
 Pi Agent    CLI Tools   Companion Apps
 (RPC mode)  (openclaw)  (macOS/iOS/Android)
```

## Core Value Proposition

- **Multi-channel AI**: Single AI agent accessible from any messaging platform
- **Plugin extensible**: Channel plugins, tool plugins, provider plugins, command plugins
- **Multi-platform**: Gateway + CLI + macOS app + iOS app + Android app
- **Self-hosted**: Runs on user's own infrastructure, no cloud dependency
- **Configurable**: Extensive JSON5 config with Zod-validated schemas

## Key Modules Map

| Module | Path | Purpose |
|--------|------|---------|
| Gateway | `src/gateway/` | WebSocket server, session routing, protocol handling |
| CLI | `src/cli/` | CLI wiring, command infrastructure |
| Commands | `src/commands/` | Individual CLI commands (agent, gateway, send, doctor, etc.) |
| Channels | `src/channels/` | Shared channel abstraction, routing, plugin system |
| Agents | `src/agents/` | Pi agent runtime, tool definitions, identity, compaction |
| Providers | `src/providers/` | Model provider implementations (Anthropic, OpenAI, Bedrock, etc.) |
| Media | `src/media/` | Media pipeline (images, audio, video processing via sharp) |
| Media Understanding | `src/media-understanding/` | Transcription and vision providers |
| Browser | `src/browser/` | Browser control via Playwright/CDP |
| Sessions | `src/sessions/` | Session state management and persistence |
| Plugin SDK | `src/plugin-sdk/` | Public API surface for extension developers |
| Config | `src/config/` | Configuration types, Zod schemas, IO, validation |
| Auto-Reply | `src/auto-reply/` | Message processing, reply pipeline, command registry |
| Routing | `src/routing/` | Session key resolution, message routing |
| Security | `src/security/` | File auditing, external content security |
| Hooks | `src/hooks/` | Hook system for lifecycle events |
| TTS | `src/tts/` | Text-to-speech pipeline |
| Memory | `src/memory/` | Vector memory (sqlite-vec) |
| Cron | `src/cron/` | Scheduled task execution |
| Infra | `src/infra/` | Infrastructure utilities, diagnostic events |

## Channel Implementations

**Core channels (in `src/`):**
- `telegram/` - Grammy bot framework
- `discord/` - @buape/carbon library
- `slack/` - @slack/bolt framework
- `signal/` - Signal protocol
- `imessage/` - iMessage bridge
- `whatsapp/` - Baileys library (WhatsApp Web)
- `web/` - Web interface
- `line/` - LINE Bot SDK

**Extension channels (in `extensions/`):**
- `msteams/` - Microsoft Teams
- `matrix/` - Matrix protocol
- `googlechat/` - Google Chat
- `bluebubbles/` - BlueBubbles (iMessage alternative)
- `voice-call/` - Voice calling
- `zalo/` / `zalouser/` - Zalo messenger
- `twitch/` - Twitch chat
- `nostr/` - Nostr protocol
- `nextcloud-talk/` - Nextcloud Talk
- `mattermost/` - Mattermost
- `tlon/` - Tlon/Urbit

## Companion Apps

| App | Path | Tech | Features |
|-----|------|------|----------|
| macOS | `apps/macos/` | Swift/SwiftUI | Menu bar, Voice Wake, Canvas, gateway control |
| iOS | `apps/ios/` | Swift/SwiftUI | Canvas, camera, Voice Wake |
| Android | `apps/android/` | Kotlin | Canvas, camera, Talk Mode |
| Shared | `apps/shared/` | Swift (OpenClawKit) | Shared framework for Apple platforms |

## Extension System

Extensions live in `extensions/*/` as pnpm workspace packages. Each has:
- `package.json` with `openclaw.extensions` array pointing to entry files
- `openclaw.plugin.json` with plugin metadata
- `index.ts` entry point that registers via the Plugin SDK API

## Related Steering Context Files

- [ARCHITECTURE.md](./ARCHITECTURE.md) - Detailed architecture and module dependencies
- [PLUGIN-SDK.md](./PLUGIN-SDK.md) - Plugin development guide
- [CHANNEL-DEVELOPMENT.md](./CHANNEL-DEVELOPMENT.md) - Building new channels
- [AGENT-TOOLS.md](./AGENT-TOOLS.md) - Agent tool development
- [VERTICALIZATION-GUIDE.md](./VERTICALIZATION-GUIDE.md) - Industry verticalization playbook
- [CODING-CONVENTIONS.md](./CODING-CONVENTIONS.md) - Code patterns and conventions
- [TESTING-GUIDE.md](./TESTING-GUIDE.md) - Testing patterns and approach
- [GATEWAY-PROTOCOL.md](./GATEWAY-PROTOCOL.md) - Gateway WebSocket protocol
- [CONFIG-SYSTEM.md](./CONFIG-SYSTEM.md) - Configuration system
