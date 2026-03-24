# @cloudflare/codemode

Expert guidance for using `@cloudflare/codemode` — the library that lets AI agents write code instead of making sequential tool calls.

## Reading Order

1. **First time?** Read this overview + Quick Start
2. **Building features?** See [Patterns](./patterns.md)
3. **API details?** See [API](./api.md)
4. **Debugging issues?** Check [Gotchas](./gotchas.md)

## Overview

Code mode transforms how AI agents interact with tools. Instead of the traditional pattern where an agent calls one tool at a time (each requiring an LLM round-trip), the agent writes a single TypeScript function that chains multiple API calls. This cuts token usage by ~81% and reduces latency dramatically.

**Traditional tool use:**
```
Agent → tool_call(getUser, {id: 1}) → result → Agent → tool_call(getOrders, {userId: 1}) → result → Agent → tool_call(formatReport, {...}) → result → Agent responds
```
(3 LLM round-trips, high token usage)

**Code mode:**
```
Agent → writes single function → executes in sandbox → Agent responds
```
(1 LLM round-trip, ~81% fewer tokens)

### How It Works

1. `createCodeTool()` takes your existing tools and creates a single `code()` tool
2. The AI agent receives the `code()` tool with TypeScript type definitions for all available APIs
3. Instead of calling tools one at a time, the agent writes a TypeScript function that uses all the APIs it needs
4. The function executes in a Dynamic Worker sandbox via `DynamicWorkerExecutor`
5. Results return to the agent in one round-trip

### Key Benefit: TypeScript > OpenAPI

Type definitions are far more token-efficient than OpenAPI specs for describing APIs to LLMs:

```typescript
// ~50 tokens — what the agent sees
interface DB {
  query(sql: string, params?: any[]): Promise<Row[]>;
  execute(sql: string, params?: any[]): Promise<void>;
}
```

vs a full OpenAPI JSON spec for the same endpoints (~500+ tokens).

## Quick Start

```typescript
import { createCodeTool, DynamicWorkerExecutor } from "@cloudflare/codemode";
import { generateText } from "ai";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const executor = new DynamicWorkerExecutor({
      loader: env.LOADER,
      globalOutbound: null,
    });

    const codemode = createCodeTool({
      tools: myTools,
      executor,
    });

    const { text } = await generateText({
      model: env.AI.binding("@cf/meta/llama-3.3-70b-instruct-fp8-fast"),
      messages: [{ role: "user", content: "Show me Alice's recent orders" }],
      tools: { codemode },
    });

    return new Response(text);
  },
};
```

## Decision Trees

### Should I use code mode?

```
Agent needs to call tools?
├─ Single tool call per turn → Standard tool use (no code mode needed)
├─ Multiple sequential tool calls → Code mode (major token savings)
├─ Complex data transformations between calls → Code mode (agent writes logic)
├─ Existing MCP server → codeMcpServer() wrapper
└─ Existing OpenAPI spec → openApiMcpServer() wrapper
```

## Resources

**Blog**: https://blog.cloudflare.com/dynamic-workers/
**npm**: `@cloudflare/codemode`

## In This Reference

- **[API](./api.md)** — createCodeTool, DynamicWorkerExecutor, codeMcpServer, openApiMcpServer
- **[Patterns](./patterns.md)** — MCP wrapping, OpenAPI integration, multi-tool composition
- **[Gotchas](./gotchas.md)** — Common errors, limits

## See Also

- **[Dynamic Workers](../dynamic-workers/README.md)** — The sandbox runtime that executes code mode functions
- **[Workers AI](../workers-ai/README.md)** — LLM inference for generating code
- **[Agents SDK](../agents-sdk/README.md)** — Building stateful AI agents
