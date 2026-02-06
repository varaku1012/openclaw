# Repository Guidelines

## Project Structure & Module Organization
Source lives in `src/` (CLI wiring in `src/cli`, commands in `src/commands`, infra in `src/infra`, media pipeline in `src/media`, web provider in `src/provider-web.ts`). Tests are colocated as `*.test.ts`, with e2e in `*.e2e.test.ts`. Docs live in `docs/`, and built output goes to `dist/`. Extensions/plugins are workspace packages under `extensions/*`. Platform apps are under `apps/*`.

## Build, Test, and Development Commands
- `pnpm install` installs dependencies (Node 22+; Bun is also supported).
- `pnpm dev` or `pnpm openclaw ...` runs the CLI in dev mode.
- `pnpm build` type-checks and builds.
- `pnpm check` runs lint/format (Oxlint/Oxfmt).
- `pnpm test` runs Vitest; `pnpm test:coverage` for coverage.

## Coding Style & Naming Conventions
Use TypeScript (ESM) with strict typing and avoid `any`. Follow existing CLI option patterns and dependency injection via `createDefaultDeps`. Keep files concise; extract helpers rather than creating “V2” copies. Naming: use “OpenClaw” for product/docs headings; use `openclaw` for CLI commands, package names, paths, and config keys.

## Testing Guidelines
Vitest is the test framework with 70% coverage thresholds. Naming: `*.test.ts` and `*.e2e.test.ts`. Live tests require explicit flags (e.g., `LIVE=1 pnpm test:live`).

## Commit & Pull Request Guidelines
Commit messages should be concise and action-oriented (e.g., `CLI: add verbose flag`). Create commits with `scripts/committer "<msg>" <files...>` to keep staging scoped. PRs should be focused, explain what/why, list testing performed, and link relevant issues. Mark AI-assisted PRs in the title or description.

## Docs & Security Notes
Docs are Mintlify-hosted: in `docs/**/*.md`, use root-relative links without `.md` (e.g., `[Config](/configuration)`). Never commit real secrets, phone numbers, or device names; use placeholders instead. Credentials live under `~/.openclaw/credentials/`, and sessions under `~/.openclaw/sessions/`.
