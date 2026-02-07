# Verticalization Blueprint

This blueprint describes how to specialize OpenClaw into domain engines (Restaurants, Marketing, Auto Dealerships) using a modular, non-forking approach.

## 1. Strategy: Composition Over Core Rewrites

- **Isolation:** Multi-agent routing via `agents.list[]` and `bindings[]`.
- **Connectivity:** Industry-specific integrations via **Extensions**.
- **Intelligence:** Domain-specific logic via **Skills**.
- **Automation:** Event-driven workflows via **Webhooks** and **Cron**.

## 2. Canonical Extension Structure

Place vertical logic in `extensions/<vertical-name>/`:

```
extensions/<vertical>/
├── package.json
├── openclaw.plugin.json   # Plugin metadata & config schema
├── index.ts               # Entry point, registers tools/hooks
├── src/
│   ├── tools/             # LLM-callable tools
│   ├── integrations/      # Domain API clients (POS, CRM, DMS)
│   ├── hooks/             # Event handlers
│   └── commands/          # Deterministic operator commands
└── skills/                # Domain-specific prompt instructions
```

## 3. Build Order for a New Vertical

1. **Topology:** Define agent IDs and channel/account bindings.
2. **Integrations:** Build API clients for the domain software.
3. **Tools:** Expose safe, minimal tool contracts (e.g., `search_menu`).
4. **Safety Flow:** Implement "Preview -> Confirm" patterns for high-risk actions.
5. **Skills:** Create `SKILL.md` files with runbooks and domain context.
6. **Automation:** Wire up async workflows (e.g., "Daily lead report").
7. **Testing:** Mock integrations and test the tool/gateway layers.

## 4. Domain Templates

### Restaurants
- **Tools:** `menu_search`, `reservation_create`, `order_preview`, `order_confirm`.
- **Integrations:** POS (Toast, Square), Reservation (OpenTable, Resy).
- **Safety:** Always require explicit user confirmation before `order_confirm`.

### Marketing Agencies
- **Tools:** `campaign_metrics`, `draft_content`, `publish_post`, `lead_lookup`.
- **Integrations:** Ad Platforms (Meta, Google), CRMs (HubSpot).
- **Safety:** Prefer "Draft-First" workflows; gate "Publish" behind admin role.

### Auto Dealerships
- **Tools:** `inventory_lookup`, `schedule_test_drive`, `generate_quote`.
- **Integrations:** DMS (DealerTrack, CDK).
- **Safety:** Label financial quotes as "Estimates" unless approved.

## 5. Deployment Pattern
For each tenant/brand:
- Dedicated `agentId`.
- Dedicated workspace/agentDir.
- Explicit `bindings` by channel and peer.
- **Never** share session state between unrelated tenants.

## 6. Prompt Layering
1. **Global:** Base safety/style.
2. **Vertical:** Industry policy and constraints.
3. **Tenant:** Brand voice and local rules.
4. **Skill:** Tool-level executable instructions.