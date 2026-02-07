# Agent Tools Development Guide

## Overview

Agent tools are functions that the AI agent can invoke during conversations. They bridge the LLM to external systems, databases, and APIs.

## Tool Architecture

```
User Message --> Agent (LLM) --> Tool Call Decision --> Tool Execution --> Result --> Agent Response
```

The agent runtime (`@mariozechner/pi-agent-core`) manages the conversation loop and tool calling protocol.

## Defining a Tool

### Basic Tool Structure

```typescript
import { Type } from "@sinclair/typebox";
import { stringEnum, optionalStringEnum } from "openclaw/plugin-sdk";

const myTool = {
  name: "my_tool_name",           // snake_case, descriptive
  description: "Clear description of what this tool does and when to use it",
  inputSchema: Type.Object({
    // Tool parameters - what the LLM provides
    action: stringEnum(["list", "create", "delete"]),
    query: Type.Optional(Type.String({ description: "Search query" })),
    limit: Type.Optional(Type.Number({ default: 10 })),
  }),

  async execute(input: { action: string; query?: string; limit?: number }, context: ToolContext) {
    switch (input.action) {
      case "list":
        return { items: await listItems(input.query, input.limit) };
      case "create":
        return { created: await createItem(input.query!) };
      case "delete":
        return { deleted: await deleteItem(input.query!) };
    }
  },
};
```

### Schema Rules (CRITICAL)

1. **Use `stringEnum()` for required string enums** - Never `Type.Union` with `Type.Literal`
2. **Use `optionalStringEnum()` for optional string enums**
3. **Add `description` to fields** for LLM understanding
4. **Keep schemas flat** - Avoid deeply nested objects
5. **Use `Type.Optional()` for non-required fields**
6. **Prefer string inputs over complex objects** - LLMs handle strings better

```typescript
// CORRECT
const schema = Type.Object({
  mode: stringEnum(["fast", "accurate"]),
  format: optionalStringEnum(["json", "text", "csv"]),
  query: Type.String({ description: "What to search for" }),
  maxResults: Type.Optional(Type.Number({ description: "Maximum results to return" })),
});

// WRONG - Don't do this
const badSchema = Type.Object({
  mode: Type.Union([Type.Literal("fast"), Type.Literal("accurate")]),  // NO!
  filters: Type.Object({                                                // Avoid nesting
    category: Type.String(),
    dateRange: Type.Object({
      start: Type.String(),
      end: Type.String(),
    }),
  }),
});
```

## Tool Context

The context parameter provides access to:

```typescript
type ToolContext = {
  config?: OpenClawConfig;          // Full configuration
  workspaceDir?: string;            // Agent workspace directory
  agentDir?: string;                // Agent-specific directory
  agentId?: string;                 // Current agent ID
  sessionKey?: string;              // Current session key
  messageChannel?: string;          // Channel (whatsapp, telegram, etc.)
  agentAccountId?: string;          // Account ID
  sandboxed?: boolean;              // Running in sandbox?
};
```

## Registering Tools

### Via Plugin SDK

```typescript
// Simple registration
api.registerTool(myTool);

// With options
api.registerTool(myTool, {
  name: "my_tool",           // Override name
  optional: true,            // Tool can be disabled
});

// Factory pattern (tool depends on context)
api.registerTool((ctx: OpenClawPluginToolContext) => {
  if (!ctx.config?.myPlugin?.enabled) return null;  // Skip if disabled

  return {
    name: "dynamic_tool",
    description: "A tool that depends on config",
    inputSchema: Type.Object({ query: Type.String() }),
    async execute(input) {
      // Use ctx.config for configuration
      return { result: "data" };
    },
  };
});

// Multiple tools from factory
api.registerTool((ctx) => [tool1, tool2, tool3]);
```

### Common Helpers from SDK

```typescript
import {
  createActionGate,    // Gate tool actions behind permissions
  jsonResult,          // Format JSON results for tool output
  readNumberParam,     // Parse number parameters safely
  readStringParam,     // Parse string parameters safely
  readReactionParams,  // Parse reaction parameters
} from "openclaw/plugin-sdk";
```

## Tool Result Patterns

### Success Result

```typescript
return { success: true, data: { id: "123", name: "Item" } };
```

### Error Result

```typescript
return { error: "Item not found", details: "No item with ID 123 exists" };
```

### List Result

```typescript
return {
  items: [
    { id: "1", name: "First" },
    { id: "2", name: "Second" },
  ],
  total: 42,
  hasMore: true,
};
```

### Action Confirmation

```typescript
return {
  action: "order_created",
  orderId: "ORD-456",
  message: "Order created successfully. Total: $42.50",
};
```

## Tool Design Best Practices

### 1. Clear, Specific Names

```typescript
// Good
name: "restaurant_menu_search"
name: "auto_inventory_lookup"
name: "marketing_campaign_metrics"

// Bad
name: "search"           // Too generic
name: "doStuff"          // Unclear
name: "handleRequest"    // Too vague
```

### 2. Descriptive Tool Description

The description is critical - it's what the LLM uses to decide when to invoke the tool:

```typescript
description: "Search the restaurant menu by category, dietary restriction, or keyword. " +
  "Use this when customers ask about menu items, prices, ingredients, or dietary options. " +
  "Returns matching items with prices and descriptions."
```

### 3. Parameter Descriptions

```typescript
inputSchema: Type.Object({
  category: optionalStringEnum(["appetizer", "entree", "dessert", "drink"], {
    description: "Filter by menu category"
  }),
  dietary: optionalStringEnum(["vegetarian", "vegan", "gluten-free", "dairy-free"], {
    description: "Filter by dietary restriction"
  }),
  query: Type.Optional(Type.String({
    description: "Free-text search term (e.g., 'pasta', 'spicy')"
  })),
}),
```

### 4. Idempotent Read Operations

Tools that only read data should be safe to call multiple times:

```typescript
async execute(input) {
  // Safe to call multiple times
  const items = await db.query(input.query);
  return { items };
}
```

### 5. Confirmable Write Operations

Dangerous operations should return a preview and require confirmation:

```typescript
async execute(input) {
  if (input.action === "delete" && !input.confirmed) {
    return {
      needsConfirmation: true,
      preview: `This will delete order ${input.orderId}. Use confirmed=true to proceed.`,
    };
  }
  await deleteOrder(input.orderId);
  return { deleted: true };
}
```

## Channel-Specific Tools

Channels can provide their own tools via `agentTools`:

```typescript
const channelPlugin: ChannelPlugin = {
  // ...
  agentTools: (ctx) => [{
    name: "telegram_set_commands",
    description: "Update the Telegram bot command menu",
    inputSchema: Type.Object({ /* ... */ }),
    async execute(input) { /* ... */ },
  }],
};
```

## Testing Tools

```typescript
import { describe, it, expect, vi } from "vitest";

describe("my_tool", () => {
  it("returns items for list action", async () => {
    const mockDb = { query: vi.fn().mockResolvedValue([{ id: "1" }]) };
    const result = await myTool.execute(
      { action: "list", query: "test" },
      { config: testConfig }
    );
    expect(result.items).toHaveLength(1);
  });

  it("handles errors gracefully", async () => {
    const result = await myTool.execute(
      { action: "delete", query: "nonexistent" },
      { config: testConfig }
    );
    expect(result.error).toBeDefined();
  });
});
```
