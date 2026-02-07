# OpenClaw API Quick Reference for AI Agents

**TL;DR**: OpenClaw achieves 8.5/10 API design quality with TypeBox-validated WebSocket RPC, adapter-based plugin system, and standardized error handling across 2,522 TypeScript files.

---

## 1-Minute Summary

**Architecture**: Gateway-centric with WebSocket RPC control plane
**Type Safety**: TypeBox schemas + AJV runtime validation (9/10)
**Consistency**: Uniform patterns across 10 API surfaces (8.5/10)
**Extensibility**: Plugin SDK for channels, tools, providers (9/10)
**Error Handling**: Standardized ErrorShape (8/10, needs more codes)

---

## Core API Contracts

### Gateway RPC Method Template

```typescript
// 1. Define schema (src/gateway/protocol/schema/my-method.ts)
import { Type } from "@sinclair/typebox";
import { NonEmptyString } from "./primitives.js";

export const MyMethodParamsSchema = Type.Object({
  required: NonEmptyString,
  optional: Type.Optional(Type.String())
}, { additionalProperties: false });

// 2. Register validator (src/gateway/protocol/index.ts)
export const validateMyMethodParams = ajv.compile(MyMethodParamsSchema);

// 3. Implement handler (src/gateway/server-methods/my-method.ts)
import type { GatewayRequestHandler } from "./types.js";
import { validateMyMethodParams, errorShape, ErrorCodes } from "../protocol/index.js";

export const myMethod: GatewayRequestHandler = async ({ params, respond }) => {
  // Validate
  if (!validateMyMethodParams(params)) {
    return respond(false, undefined, errorShape(
      ErrorCodes.INVALID_REQUEST,
      formatValidationErrors(validateMyMethodParams.errors)
    ));
  }
  
  try {
    // Execute
    const result = await doSomething(params.required);
    respond(true, { result });
  } catch (err) {
    respond(false, undefined, errorShape(
      ErrorCodes.INTERNAL_ERROR,
      String(err),
      { retryable: true }
    ));
  }
};

// 4. Register method (src/gateway/server-methods-list.ts)
export const methods = {
  "my.method": myMethod,
  // ...
};
```

---

### Channel Plugin Template

```typescript
// extensions/my-channel/src/channel.ts
import type { ChannelPlugin } from "openclaw/plugin-sdk";

export const myChannel: ChannelPlugin = {
  id: "my-channel",
  meta: {
    id: "my-channel",
    label: "My Channel",
    selectionLabel: "My Channel (description)",
    docsPath: "/channels/my-channel",
    blurb: "Short description"
  },
  capabilities: {
    chatTypes: ["dm", "group"],
    reactions: true,
    media: true
  },
  
  // REQUIRED: Config adapter
  config: {
    listAccountIds: (cfg) => Object.keys(cfg.myChannel?.accounts ?? {}),
    resolveAccount: (cfg, accountId) => {
      const id = accountId ?? "default";
      return cfg.myChannel?.accounts?.[id] ?? {};
    }
  },
  
  // OPTIONAL: Gateway adapter (for WebSocket channels)
  gateway: {
    startAccount: async (ctx) => {
      // Start channel connection
      const client = await connectToChannel(ctx.account);
      
      // Listen for messages
      client.on("message", (msg) => {
        emitDiagnosticEvent({
          type: "message.received",
          channel: "my-channel",
          payload: { from: msg.from, text: msg.text }
        });
      });
      
      // Update status
      ctx.setStatus({
        accountId: ctx.accountId,
        connected: true,
        lastConnectedAt: Date.now()
      });
      
      return client;
    },
    
    stopAccount: async (ctx) => {
      // Disconnect gracefully
    }
  },
  
  // OPTIONAL: Outbound adapter
  outbound: {
    deliveryMode: "direct",
    sendText: async (ctx) => {
      const result = await sendMessage(ctx.to, ctx.text);
      return {
        ok: true,
        messageId: result.id,
        timestamp: result.timestamp
      };
    }
  }
};
```

---

### Agent Tool Template

```typescript
// src/agents/tools/my-tool.ts
import type { AgentTool } from "@mariozechner/pi-agent-core";
import { Type } from "@sinclair/typebox";
import { stringEnum, readStringParam, jsonResult } from "./common.js";

const MyToolSchema = Type.Object({
  action: stringEnum(["create", "update", "delete"]),  // NOT Type.Union!
  target: Type.String({ description: "Target identifier" }),
  options: Type.Optional(Type.Object({
    dryRun: Type.Boolean({ default: false })
  }))
});

export const myTool: AgentTool<typeof MyToolSchema, unknown> = {
  name: "my_tool",
  description: "Brief description for LLM understanding",
  inputSchema: MyToolSchema,
  
  executor: async (params) => {
    // Type-safe parameter access (validated by TypeBox)
    const action = params.action;  // "create" | "update" | "delete"
    const target = params.target;
    const dryRun = params.options?.dryRun ?? false;
    
    if (dryRun) {
      return jsonResult({ 
        message: `Would ${action} ${target}`,
        dryRun: true 
      });
    }
    
    // Execute action
    const result = await performAction(action, target);
    
    return jsonResult({
      success: true,
      action,
      target,
      result
    });
  }
};
```

---

## Critical Rules

### DO

1. **Always validate inputs with TypeBox**:
   ```typescript
   if (!validateParams(params)) {
     return errorShape(ErrorCodes.INVALID_REQUEST, formatValidationErrors(validateParams.errors));
   }
   ```

2. **Use stringEnum for tool schemas** (NOT Type.Union):
   ```typescript
   stringEnum(["option1", "option2"])  // GOOD
   Type.Union([Type.Literal("option1"), ...])  // BAD (breaks Gemini)
   ```

3. **Return standardized ErrorShape**:
   ```typescript
   errorShape(ErrorCodes.MY_ERROR, "Human message", { 
     details: {...},
     retryable: true,
     retryAfterMs: 5000
   })
   ```

4. **Support AbortSignal**:
   ```typescript
   async function myMethod(ctx: { abortSignal: AbortSignal }) {
     if (ctx.abortSignal.aborted) {
       throw new Error("Aborted");
     }
     // ...
   }
   ```

5. **Immutable config updates**:
   ```typescript
   return { ...cfg, myChannel: { ...cfg.myChannel, enabled: true } };  // GOOD
   cfg.myChannel.enabled = true; return cfg;  // BAD
   ```

### DON'T

1. ❌ Use plain Error objects (no error codes)
2. ❌ Skip schema validation (always validate inputs)
3. ❌ Return `Promise<unknown>` (always type responses)
4. ❌ Mix naming conventions (stick to camelCase for params)
5. ❌ Use Type.Union in tool schemas (breaks Gemini)
6. ❌ Forget AbortSignal checks (causes hanging requests)
7. ❌ Mutate config objects (always return new)
8. ❌ Use magic strings for error codes (use ErrorCodes constants)

---

## Common Patterns

### Session Key Format
```typescript
"<channel>:<accountId>:<conversationId>"
"whatsapp:default:+1234567890"
"telegram:personal:@username"
```

### Error Response
```typescript
{
  code: "ERROR_CODE",
  message: "Human-readable description",
  details?: unknown,
  retryable?: boolean,
  retryAfterMs?: number
}
```

### Gateway Frame Types
```typescript
// Request (client → server)
{ type: "req", id: "req_123", method: "agent", params: {...} }

// Response (server → client)
{ type: "res", id: "req_123", ok: true, payload: {...} }

// Event (server → client, unsolicited)
{ type: "event", event: "agent", payload: {...}, seq: 0 }
```

### Tool Result
```typescript
{
  content: [
    { type: "text", text: "Human-readable output" },
    { type: "image", data: "base64...", mimeType: "image/png" }
  ],
  details: { /* machine-readable */ }
}
```

---

## File Locations

**Gateway Protocol**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/src/gateway/protocol/`
**Channel Plugins**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/extensions/*/src/channel.ts`
**Agent Tools**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/src/agents/tools/`
**Plugin SDK**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/src/plugin-sdk/index.ts`
**Server Methods**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/src/gateway/server-methods/`

---

## Best Examples to Study

**Gateway Method**: `src/gateway/server-methods/agent.ts`
**Channel Plugin**: `extensions/bluebubbles/src/channel.ts`
**Agent Tool**: `src/agents/tools/browser-tool.ts`
**Error Handling**: `src/gateway/protocol/index.ts` (formatValidationErrors)
**Type Safety**: `src/gateway/protocol/schema/frames.ts`

---

## When to Consult Full Assessment

1. Designing new API surfaces (read sections 1-10)
2. Understanding protocol versioning (section 1)
3. Adding new channel plugins (section 2)
4. Creating provider plugins (section 5)
5. Improving error handling (section 9)
6. REST endpoint design (section 10)

**Full Document**: `/mnt/c/Users/rajesh vankayalapati/repos/openclaw/.claude/memory/api-design/API_QUALITY_ASSESSMENT.md` (1,765 lines, 50 KB)
