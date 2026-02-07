# OpenClaw Verticalization Guide

## Purpose

This guide provides a playbook for verticalizing the OpenClaw engine to specific industries. The engine's plugin architecture makes it highly adaptable for domain-specific deployments.

## Verticalization Strategy

Each industry vertical is implemented as a **plugin package** (or set of plugins) that extends the core engine with:

1. **Domain-specific agent tools** - Industry-specific actions the AI can perform
2. **Custom commands** - Slash commands for common operations
3. **Webhook integrations** - HTTP endpoints for industry platforms
4. **Custom configuration** - Industry-specific settings
5. **Agent context** - Domain knowledge and behavior rules
6. **Background services** - Scheduled tasks, monitoring

## Architecture Pattern for Verticals

```
extensions/<industry>/
  package.json
  openclaw.plugin.json
  index.ts                    # Main plugin entry
  src/
    tools/                    # Agent tools for this vertical
      booking-tool.ts
      inventory-tool.ts
      analytics-tool.ts
    commands/                 # Chat commands
      status-command.ts
    hooks/                    # Lifecycle hooks
      message-enrichment.ts
    services/                 # Background services
      sync-service.ts
    integrations/             # External API clients
      pos-client.ts
      crm-client.ts
    config/                   # Configuration schema
      schema.ts
    prompts/                  # Agent system prompt additions
      identity.ts
```

## Creating a Vertical Plugin

### Step 1: Scaffold the Extension

```bash
mkdir -p extensions/<industry>/src/{tools,commands,hooks,services,integrations,config,prompts}
```

### Step 2: Define package.json

```json
{
  "name": "@openclaw/<industry>",
  "version": "1.0.0",
  "type": "module",
  "devDependencies": {
    "openclaw": "workspace:*"
  },
  "dependencies": {
    // Industry-specific dependencies ONLY
  },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

### Step 3: Implement the Plugin

```typescript
// extensions/<industry>/index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { registerTools } from "./src/tools/index.js";
import { registerCommands } from "./src/commands/index.js";
import { registerHooks } from "./src/hooks/index.js";
import { registerServices } from "./src/services/index.js";

export default {
  id: "openclaw-<industry>",
  name: "OpenClaw for <Industry>",
  version: "1.0.0",
  configSchema: industryConfigSchema,

  register(api: OpenClawPluginApi) {
    registerTools(api);
    registerCommands(api);
    registerHooks(api);
    registerServices(api);

    // Inject domain context into agent prompts
    api.on("before_agent_start", async (event, ctx) => {
      return {
        prependContext: buildIndustryContext(api.pluginConfig),
      };
    });
  }
};
```

---

## Industry Vertical: Restaurants

### Use Cases
- Order management (take orders, modify, cancel)
- Reservation handling (book, confirm, reschedule, cancel)
- Menu inquiries (dietary info, prices, specials)
- Kitchen communication (order status, prep times)
- Customer feedback collection
- Daily specials announcement to regular customers

### Agent Tools to Implement

| Tool | Description | Integration |
|------|-------------|-------------|
| `restaurant_menu` | Query menu items, prices, dietary info | POS/Menu database |
| `restaurant_order` | Create/modify/cancel orders | POS system (Toast, Square, Clover) |
| `restaurant_reservation` | Book/modify/cancel reservations | OpenTable, Resy, or custom |
| `restaurant_kitchen_status` | Check order prep status | Kitchen Display System |
| `restaurant_inventory` | Check ingredient availability | Inventory management |
| `restaurant_feedback` | Collect and route customer feedback | CRM/Feedback system |
| `restaurant_specials` | Manage daily specials | Menu management |
| `restaurant_hours` | Business hours and holiday schedule | Config/Calendar |
| `restaurant_loyalty` | Check/apply loyalty points | Loyalty program |

### Commands

| Command | Description |
|---------|-------------|
| `/menu` | Show current menu |
| `/reserve` | Quick reservation |
| `/orderstatus` | Check order status |
| `/specials` | Today's specials |
| `/hours` | Business hours |
| `/feedback` | Submit feedback |

### Configuration Schema

```typescript
type RestaurantConfig = {
  restaurant: {
    name: string;
    timezone: string;
    pos: {
      provider: "toast" | "square" | "clover" | "custom";
      apiKey: string;
      locationId: string;
    };
    reservations: {
      provider: "opentable" | "resy" | "builtin";
      maxPartySize: number;
      slotDurationMinutes: number;
    };
    hours: Record<string, { open: string; close: string }>;
    loyalty?: { enabled: boolean; provider: string };
  };
};
```

### Agent Context

```
You are the AI assistant for [Restaurant Name]. You help customers with:
- Taking and modifying food orders
- Making and managing reservations
- Answering menu questions (ingredients, allergens, prices)
- Providing order status updates
- Collecting feedback

Rules:
- Always confirm orders before submitting
- Check dietary restrictions before recommending
- For parties > 8, suggest calling the restaurant directly
- Be warm and conversational but efficient
```

---

## Industry Vertical: Marketing Agencies

### Use Cases
- Campaign management (create, monitor, report)
- Content generation and scheduling
- Social media management
- Client communication and reporting
- Analytics dashboard queries
- Lead management and nurturing
- Invoice and project status

### Agent Tools to Implement

| Tool | Description | Integration |
|------|-------------|-------------|
| `marketing_campaign` | Create/manage campaigns | Ad platforms (Meta, Google) |
| `marketing_analytics` | Query campaign metrics | Google Analytics, Meta Insights |
| `marketing_content` | Generate/schedule content | Buffer, Hootsuite, or custom |
| `marketing_social` | Post to social platforms | Social media APIs |
| `marketing_leads` | Manage leads and pipeline | CRM (HubSpot, Salesforce) |
| `marketing_report` | Generate client reports | Reporting engine |
| `marketing_calendar` | Content calendar management | Calendar/Project tool |
| `marketing_seo` | SEO analysis and suggestions | SEMrush, Ahrefs API |
| `marketing_email` | Email campaign management | Mailchimp, SendGrid |
| `marketing_budget` | Budget tracking and allocation | Financial system |

### Commands

| Command | Description |
|---------|-------------|
| `/campaign` | Campaign overview |
| `/report` | Generate client report |
| `/analytics` | Quick analytics summary |
| `/schedule` | Content schedule |
| `/leads` | Lead pipeline status |
| `/budget` | Budget utilization |

### Agent Context

```
You are an AI marketing assistant for [Agency Name]. You help with:
- Campaign creation and monitoring across platforms
- Content ideation, generation, and scheduling
- Analytics reporting and insights
- Lead management and nurturing
- Client communication

Rules:
- Always include data to support recommendations
- Follow brand guidelines when generating content
- Flag budget concerns proactively
- Never post content without explicit approval
```

---

## Industry Vertical: Automobile Dealerships

### Use Cases
- Vehicle inventory inquiries
- Test drive scheduling
- Finance/lease quote generation
- Service appointment booking
- Trade-in value estimation
- Customer follow-up and nurturing
- Parts ordering

### Agent Tools to Implement

| Tool | Description | Integration |
|------|-------------|-------------|
| `auto_inventory` | Search vehicle inventory | DMS (CDK, Reynolds, DealerSocket) |
| `auto_testdrive` | Schedule test drives | Calendar/CRM |
| `auto_finance` | Generate finance/lease quotes | Finance calculator |
| `auto_tradein` | Estimate trade-in value | KBB, NADA, Black Book API |
| `auto_service` | Book service appointments | Service scheduler |
| `auto_parts` | Check parts availability | Parts inventory |
| `auto_customer` | Customer profile and history | CRM |
| `auto_compare` | Compare vehicles side-by-side | Inventory + specs database |
| `auto_insurance` | Insurance quote estimation | Insurance partners |
| `auto_recall` | Check vehicle recalls | NHTSA API |

### Commands

| Command | Description |
|---------|-------------|
| `/inventory` | Search vehicles |
| `/testdrive` | Schedule test drive |
| `/quote` | Get finance quote |
| `/tradein` | Trade-in estimate |
| `/service` | Book service |
| `/compare` | Compare vehicles |

### Agent Context

```
You are the AI sales and service assistant for [Dealership Name]. You help with:
- Finding the right vehicle from our inventory
- Scheduling test drives
- Providing finance and lease options
- Booking service appointments
- Answering vehicle specification questions

Rules:
- Never quote final prices - always say "starting from" or "estimated"
- Always encourage in-person visits for final decisions
- Be transparent about vehicle condition and history
- For finance, always mention "subject to credit approval"
- Collect customer contact info for follow-up when appropriate
```

---

## Standard Build Order for a New Vertical

Follow this sequence — each step builds on the previous:

1. **Define tenant boundaries** — which channels/accounts map to which agent IDs (`agents.list[]`, `bindings[]`)
2. **Model domain API clients** — under `extensions/<vertical>/src/integrations/`
3. **Expose read-only tools first** — safe, minimal tool contracts in `extensions/<vertical>/src/tools/`
4. **Add write tools with preview/confirm split** — never combine preview and commit in one tool
5. **Add skill instructions** — teach the agent correct usage sequence and confirmation flows
6. **Wire webhook/cron** — only after manual paths are stable
7. **Add tests** — at tool, integration-mock, and gateway method layers

## Suggested Milestones per Vertical

| Milestone | Scope |
|-----------|-------|
| **Foundation** | Integration clients + read-only tools + skills |
| **Controlled Writes** | Preview + confirm split actions |
| **Automation** | Webhook and cron flows |
| **Expansion** | Multi-agent tenant routing and reporting |

## Multi-Tenant Deployment Pattern

Each tenant/brand requires:
- Dedicated `agentId` in `agents.list[]`
- Dedicated `workspace` directory
- Dedicated `agentDir` for session storage
- Explicit `bindings[]` mapping channel/account/peer to agent
- Session collision impossible when agent IDs are unique per tenant

## Prompt/Skill Layering Pattern

Layer prompts from broad to narrow:

1. **Global safety and style** — base system prompt
2. **Vertical policy** — domain constraints (e.g., "never confirm orders without inventory check")
3. **Tenant policy** — brand voice, local business rules
4. **Task skill instructions** — tool-level runbooks in `SKILL.md` files

Keep system prompts compact. Place long runbooks in `SKILL.md` files.

## Operational Readiness Checklist

Before rollout for any vertical:

1. [ ] DM/group access policy is locked down for production channels
2. [ ] Tool allowlists and denylists are properly configured
3. [ ] High-risk actions require explicit user confirmation (preview/confirm split)
4. [ ] Plugin config schema and UI hints are complete
5. [ ] Cron/webhook flows are idempotent and observable
6. [ ] Tests cover all commit-path (write) actions

## Anti-Patterns to Avoid

- Putting domain logic directly into core gateway handlers (use plugin tools)
- Using one monolithic tool for preview + commit side effects
- Relying only on prompt instructions for irreversible operations (use tool-level gating)
- Shipping vertical features without explicit routing isolation
- Hardcoding tenant config (use `pluginConfig` with schema validation)

## Implementation Checklist for Any Vertical

1. [ ] Create extension package structure
2. [ ] Define plugin metadata (`openclaw.plugin.json`)
3. [ ] Implement configuration schema (Zod)
4. [ ] Build integration clients for industry platforms
5. [ ] Implement agent tools with proper TypeBox schemas
6. [ ] Register chat commands for common operations
7. [ ] Add `before_agent_start` hook for domain context injection
8. [ ] Add `message_received` hook for message enrichment (if needed)
9. [ ] Implement background sync services (if needed)
10. [ ] Register webhook endpoints (if needed)
11. [ ] Write unit tests for tools and commands
12. [ ] Write integration tests for external API clients
13. [ ] Add configuration documentation
14. [ ] Test across multiple channels (WhatsApp, Telegram, Slack, etc.)

## Key Patterns for Vertical Development

### Tool Error Handling

```typescript
async execute(input) {
  try {
    const result = await externalApi.call(input);
    return { success: true, data: result };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : "Unknown error"
    };
  }
}
```

### Configuration Access

```typescript
api.on("before_agent_start", async (event, ctx) => {
  const industryConfig = api.pluginConfig as IndustryConfig;
  // Use config to customize behavior
});
```

### Multi-Channel Awareness

Tools should work across all channels. Avoid channel-specific assumptions:

```typescript
async execute(input, context) {
  // context.messageChannel tells you which channel (whatsapp, telegram, etc.)
  // Adapt response format if needed, but keep logic universal
  const result = await processOrder(input);

  // Return plain text - the channel adapter handles formatting
  return { text: formatOrderConfirmation(result) };
}
```

### Background Data Sync

```typescript
api.registerService({
  id: "inventory-sync",
  start: async (ctx) => {
    // ctx.stateDir - store sync state here
    // ctx.logger - use for logging
    setInterval(async () => {
      await syncInventory(ctx);
    }, 5 * 60 * 1000); // Every 5 minutes
  },
  stop: async () => {
    // Cleanup intervals
  }
});
```
