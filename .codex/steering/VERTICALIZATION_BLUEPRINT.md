# Verticalization Blueprint

Last validated: 2026-02-06

This blueprint describes how to specialize OpenClaw into domain engines (restaurants, marketing agencies, auto dealerships) without forking core architecture.

## 1) Guiding Strategy

Use composition over core rewrites:

- Tenant isolation: multi-agent routing (`agents.list[]`, `bindings[]`)
- Domain actions: plugin tools + plugin commands (`extensions/<vertical>`)
- Domain behavior: skills (`skills/` or plugin-shipped skills)
- Event automation: webhooks + cron

Default target is one vertical package per domain plus optional shared utility plugins.

## 2) Canonical Vertical Package Shape

`extensions/<vertical>/`

- `extensions/<vertical>/package.json`
- `extensions/<vertical>/openclaw.plugin.json`
- `extensions/<vertical>/index.ts`
- `extensions/<vertical>/src/tools/`
- `extensions/<vertical>/src/commands/`
- `extensions/<vertical>/src/integrations/`
- `extensions/<vertical>/src/hooks/`
- `extensions/<vertical>/src/services/`
- `extensions/<vertical>/skills/` (optional, if shipping domain skills)

## 3) Standard Build Order for a New Vertical

1. Define tenant boundaries and agent topology (which channels/accounts map to which agent ids).
2. Model the domain API clients under `extensions/<vertical>/src/integrations/`.
3. Expose safe, minimal tool contracts in `extensions/<vertical>/src/tools/`.
4. Add deterministic operator commands in `extensions/<vertical>/src/commands/` for high-risk actions.
5. Add skill instructions that teach confirmation flows and domain language.
6. Wire webhook/cron ingestion if the vertical requires async workflows.
7. Add tests at tool, integration-mock, and gateway method layers.

## 4) Domain Templates

## Restaurants

Core domain tools:

- `restaurant_menu_search`
- `restaurant_reservation_create`
- `restaurant_reservation_update`
- `restaurant_order_create_preview`
- `restaurant_order_confirm`
- `restaurant_order_status`

Likely integrations:

- POS
- Reservation platform
- Delivery provider

Safety pattern:

- Separate preview and commit tools for order placement.
- Require explicit user confirmation phrase before commit tool call.

## Marketing Agencies

Core domain tools:

- `campaign_create_draft`
- `campaign_publish`
- `campaign_metrics_get`
- `content_calendar_update`
- `lead_status_update`

Likely integrations:

- Ad platform APIs
- Analytics provider
- CRM
- Content scheduler

Safety pattern:

- Keep publish operations gated and auditable.
- Prefer draft-first workflows.

## Automobile Dealerships

Core domain tools:

- `inventory_search`
- `test_drive_schedule`
- `finance_quote_generate`
- `trade_in_estimate`
- `service_booking`

Likely integrations:

- DMS
- Service scheduler
- Pricing/valuation APIs

Safety pattern:

- Mark generated finance quotes as estimates unless a final approval path is invoked.

## 5) Multi-Tenant Deployment Pattern

Recommended minimum for each tenant/brand:

- Dedicated `agentId`
- Dedicated workspace
- Dedicated `agentDir`
- Explicit `bindings` by channel/account/peer

Example direction:

- `agents.list[]` entries: `restaurant-main`, `agency-main`, `dealership-main`
- Bind channels and accounts explicitly via `bindings[]`
- Keep session collisions impossible by avoiding shared agent ids for unrelated tenants

## 6) Prompt/Skill Layering Pattern

Layer prompts from broad to narrow:

1. Global safety and style
2. Vertical policy (domain constraints)
3. Tenant policy (brand voice, local business rules)
4. Task skill instructions (tool-level runbooks)

Place long runbooks in `SKILL.md` files; keep system prompt compact.

## 7) Operational Readiness Checklist

Before rollout for any vertical:

1. Verify DM/group access policy is locked down for production channels.
2. Verify tool allowlists and denylist boundaries.
3. Verify high-risk actions require explicit confirmation.
4. Verify plugin config schema and UI hints are complete.
5. Verify cron/webhook flows are idempotent and observable.
6. Verify tests cover commit-path actions.

## 8) Anti-Patterns to Avoid

- Putting domain logic directly into core gateway handlers unless needed by all users.
- Using one monolithic tool for preview + commit side effects.
- Relying only on prompt instructions for irreversible operations.
- Shipping vertical features without explicit routing isolation.

## 9) Suggested Milestones per Vertical

1. Foundation: integrations + read-only tools + skills.
2. Controlled writes: preview + confirm split actions.
3. Automation: webhook and cron flows.
4. Expansion: multi-agent tenant routing and reporting.
