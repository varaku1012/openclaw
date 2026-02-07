# Development Workflow

## Environment Setup
*   **Node.js:** >= 22.12.0
*   **Package Manager:** `pnpm` (v10.23.0 strict)
*   **Language:** TypeScript (Strict Mode)

## Key Commands

### Build
*   `pnpm install`: Install dependencies.
*   `pnpm build`: Type-check and build the core and extensions.
*   `pnpm ui:build`: Build the frontend UI.
*   `pnpm prepack`: Full build (Core + UI).

### Run
*   `pnpm dev`: Run the CLI/Gateway in development mode (hot reload).
*   `pnpm gateway:watch`: Watch mode specifically for the Gateway.

### Test
*   `pnpm test`: Run unit tests (Vitest).
*   `pnpm test:e2e`: Run end-to-end tests.
*   **Conventions:**
    *   Colocate tests with source files: `feature.ts` -> `feature.test.ts`.
    *   E2E tests: `feature.e2e.test.ts`.

### Code Quality
*   `pnpm lint`: Run `oxlint`.
*   `pnpm format`: Run `oxfmt` (check).
*   `pnpm format:fix`: Fix formatting issues.
*   `pnpm check`: Run type-check, lint, and format check.

## Creating New Features
1.  **Understand the Domain:** Is it a Gateway feature, an Agent feature, or an Extension?
2.  **Follow Patterns:** Look at existing code in `src/`. Copy-paste-modify is encouraged for boilerplate (e.g., new Channel adapters).
3.  **Add Tests:** Every new feature MUST have tests. Use `test-helpers` in `src/gateway/test-helpers.ts` or `src/agents/test-helpers/`.
4.  **Lint & Format:** Always run `pnpm check` before finishing.

## File Naming
*   Use `kebab-case` for files and directories.
*   Use `PascalCase` for Classes.
*   Use `camelCase` for functions and variables.
