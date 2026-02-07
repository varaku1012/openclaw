# Repository Map

## Core (`src/`)
*   `src/gateway/`: **Gateway Server**. The entry point and control plane.
    *   `server.ts`: Main entry point.
    *   `server-sessions.ts`: Session management.
*   `src/agents/`: **Agent Runtime**.
    *   `pi-embedded-runner.ts`: The main loop for executing agents.
    *   `tools/`: Internal tools.
*   `src/channels/`: **Core Channels**.
    *   `whatsapp/`, `telegram/`, `slack/`: Platform-specific implementations.
*   `src/plugin-sdk/`: **Plugin SDK**.
    *   `index.ts`: Exports all interfaces required to build extensions.
*   `src/cli/`: **CLI Entry**.
    *   `entry.ts`: Command-line argument parsing.

## Extensions (`extensions/`)
*   Plugins that add channels or capabilities.
    *   `googlechat/`: Example channel extension.
    *   `voice-call/`: Example capability extension.

## Skills (`skills/`)
*   Library of capabilities available to agents.
    *   `food-order/`, `local-places/`: Examples of domain-specific skills.

## Configuration
*   `package.json`: Dependencies and scripts.
*   `pnpm-workspace.yaml`: Monorepo definitions.
*   `AGENTS.md`: Agent guidelines.
*   `GEMINI.md`: Context for Gemini (you are reading a derivative of this).
