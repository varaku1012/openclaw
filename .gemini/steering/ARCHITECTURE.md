# OpenClaw Architecture

## High-Level Overview

OpenClaw is a personal AI assistant platform designed to run locally on user devices. It acts as a central **Gateway** that connects various **Messaging Channels** (WhatsApp, Telegram, Slack, etc.) to **AI Agents**.

The architecture is composed of three main layers:
1.  **Gateway (The Nervous System):** Manages connections, message routing, sessions, and the WebSocket control plane.
2.  **Agents (The Brain):** The runtime environment that executes AI models, manages context, and calls tools.
3.  **Extensions & Skills (The Limbs):** Plugins that add new channels, capabilities, and domain-specific tools.

## Core Components

### 1. Gateway (`src/gateway`)
The Gateway is the server component that runs locally.
-   **Entry Point:** `src/gateway/server.ts` (starts via `startGatewayServer`).
-   **Responsibilities:**
    -   **Protocol:** Implements the OpenClaw Protocol for client-server communication.
    -   **Session Management:** Handles user sessions (`src/gateway/server-sessions.ts`).
    -   **Message Routing:** Routes messages from Channels to Agents and back.
    -   **Discovery:** Manages discovery of local nodes and capabilities.

### 2. Agents (`src/agents`)
The Agent Runtime executes the "cognitive loop" of the AI.
-   **Runner:** `src/agents/pi-embedded-runner.ts` is the core loop that manages the "Think -> Tool -> Action" cycle.
-   **Context:** Manages the context window, history, and system prompts.
-   **Tools:** Exposes internal tools and loaded Skills to the LLM.

### 3. Channels (`src/channels`, `extensions/*`)
Channels are adapters that connect external messaging platforms to the Gateway.
-   **Core Channels:** WhatsApp (Web), Telegram, Slack, etc.
-   **Interface:** Channels implement a set of adapters (Auth, Setup, Messaging) defined in `src/plugin-sdk`.
-   **Extensions:** New channels are best implemented as extensions (e.g., `extensions/googlechat`).

### 4. Skills (`skills/`)
Skills are packages of tools and prompts that give Agents specific capabilities.
-   **Structure:** A directory containing a definition (likely `SKILL.md` or similar) and executable code/scripts.
-   **Usage:** Skills are loaded by the Agent Runtime and exposed as tools to the LLM.

## Data Flow
1.  **Inbound:** User sends a message on a Channel (e.g., Telegram).
2.  **Routing:** Channel Adapter receives the message and forwards it to the Gateway.
3.  **Processing:** Gateway routes the message to the active Agent Session.
4.  **Cognition:** Agent receives the message, consults its System Prompt and History, and generates a response or Tool Call.
5.  **Execution:** If a Tool is called, the Agent Runtime executes it (e.g., "Look up menu").
6.  **Outbound:** Agent sends the final text response back to the Gateway.
7.  **Delivery:** Gateway forwards the response to the Channel Adapter, which sends it to the user.
