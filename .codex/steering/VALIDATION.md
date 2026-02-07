# Steering Validation

Validated on: 2026-02-07

This report captures the verification pass for `.codex/steering/`.

## Checks Run

1. File presence check for required steering pack files.
2. Path-reference validation across backticked references in steering markdown.
3. Manual consistency review against current source structure and docs.

## Path Validation Policy

- Template paths using placeholders (for example `extensions/<vertical>/...`) are allowed.
- Wildcard patterns (for example `src/gateway/protocol/schema/*.ts`) are allowed.
- Generated build-output paths (for example dist/) are allowed even if not present locally.
- Intentional drift examples are allowed when clearly documented.

## Results

- Steering files present:
  - `README.md`
  - `SYSTEM_MAP.md`
  - `ENGINEERING_GUARDRAILS.md`
  - `EXTENSION_SURFACES.md`
  - `VERTICALIZATION_BLUEPRINT.md`
  - `AI_AGENT_WORKFLOW.md`
  - `VALIDATION.md`
- Path-reference scan summary:
  - Checked references: 202
  - Skipped template placeholders: 29
  - Skipped wildcard patterns: 20
  - Skipped command-like snippets: 2
  - Skipped example-only refs: 5
  - Actionable missing references: 0

## Cross-Team Synthesis

Reviewed and compared `.gemini/steering/` and `.claude/steering/` against source. Adopted only source-verified guidance:

- Added tool schema portability rules (`stringEnum`/`optionalStringEnum`, no top-level union schemas) with canonical references.
- Added protocol regeneration requirements for gateway method changes (`pnpm protocol:gen`, `pnpm protocol:gen:swift` when needed).
- Added plugin runtime API coverage (hooks/services/http/gateway/command registration).
- Added resilient integration patterns grounded in existing modules (auth profile cooldown/order, guarded fetch + SSRF policy, shared backoff helpers, lock-based file writes).
- Added practical test stability patterns (`withTempHome`, `pollUntil`, deterministic port blocks, test plugin registry stubs).

## Intentional Exceptions

- src/provider-web.ts in `README.md` is intentionally referenced as a drift callout because `AGENTS.md` still mentions it while runtime has moved to `src/web/` + `src/channel-web.ts`.
- dist/ in `ENGINEERING_GUARDRAILS.md` may be absent before build and is treated as a generated-output path.
