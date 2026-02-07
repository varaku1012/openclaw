# OpenClaw System Architecture

OpenClaw is a modular AI Assistant platform composed of a Gateway (Nervous System), Agents (Brain), and Extensions (Limbs).

## 1. The Three-Layer Stack

### Gateway (`src/gateway`)
The central control plane and WebSocket server.
-   **Routing:** Maps incoming messages from peers to specific Agent sessions.
-   **Security:** Enforces auth scopes and DM/Group policies.
-   **State:** Manages the active run loop and streaming delivery.

### Agents (`src/agents`)
The execution environment for LLMs.
-   **Runner:** The Think-Tool-Act cycle.
-   **Memory:** Context management and compaction.
-   **Identity:** Personas, avatars, and custom system prompts.

### Extensions & Channels (`extensions/`, `src/channels`)
The connectivity layer.
-   **Channels:** Adapters for WhatsApp, Telegram, etc.
-   **Plugins:** Background services, custom HTTP routes, and additional tools.

## 2. Core Entities (Domain Model)

-   **Agent:** A unique persona ID with specific config (model, tools, workspace).
-   **Session:** A deterministic conversation ID: `agent:{id}:{scope}`.
-   **Binding:** A configuration rule mapping a `(channel, account, peer)` tuple to an `agentId`.
-   **Skill:** A bundle of tools and system-prompt additions.

## 3. Message Processing Pipeline

1.  **Inbound:** Channel receives raw message.
2.  **Normalization:** Message is converted to a standard `Envelope`.
3.  **Security Gate:** DM/Group policy and allowlist checks.
4.  **Routing:** `resolve-route.ts` identifies the target `agentId` and `sessionKey`.
5.  **Agent Invocation:** `agent` RPC method is called.
6.  **Cognition:** LLM processes context, history, and tools.
7.  **Outbound:** Response is delivered via the outbound delivery service.

## 4. Key Directory Map

-   `src/gateway/`: Server, protocol, and RPC methods.
-   `src/agents/`: Model selection, runners, and context logic.
-   `src/channels/`: Built-in messaging platform adapters.
-   `src/plugin-sdk/`: Public interfaces for building extensions.
-   `extensions/`: First-party and community plugins.
-   `skills/`: Bundled agent capabilities.
-   `docs/`: Mintlify-based documentation.