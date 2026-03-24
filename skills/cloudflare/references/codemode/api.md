# @cloudflare/codemode API

## createCodeTool(options)

Creates a single `code` tool from a set of existing tools. The AI agent receives TypeScript type definitions and writes functions that call these tools.

```typescript
import { createCodeTool } from "@cloudflare/codemode";

const codemode = createCodeTool({
  tools: myTools,
  executor,
});
```

**Parameters:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `tools` | `Tool[]` | Yes | Array of tool definitions (functions + type info) |
| `executor` | `DynamicWorkerExecutor` | Yes | Executor that runs the generated code |

**Returns:** A single tool object compatible with Vercel AI SDK, Anthropic SDK, etc. The tool exposes all input tools' APIs as TypeScript interfaces to the LLM.

## DynamicWorkerExecutor

Wraps `env.LOADER` to construct sandboxes with code normalization for AI-generated code.

```typescript
import { DynamicWorkerExecutor } from "@cloudflare/codemode";

const executor = new DynamicWorkerExecutor({
  loader: env.LOADER,
  globalOutbound: null,
});
```

**Constructor options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `loader` | `DynamicWorkerLoader` | Yes | The `env.LOADER` binding |
| `globalOutbound` | `null \| Function` | No | HTTP control for sandboxes (default: null) |

### executor.execute(code, providers?)

Execute a code string in a Dynamic Worker sandbox with optional tool providers.

```typescript
const result = await executor.execute(`
  async () => {
    const users = await tools.db.query("SELECT * FROM users LIMIT 10");
    return users.map(u => u.name);
  }
`, [myToolProvider]);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `code` | `string` | TypeScript/JavaScript function string |
| `providers` | `Provider[]` | Optional array of tool/API providers |

## codeMcpServer(mcpServer)

Wraps an existing MCP (Model Context Protocol) server with a single `code()` tool. Instead of the agent calling MCP tools individually, it writes code that calls them.

```typescript
import { codeMcpServer } from "@cloudflare/codemode";

const wrappedServer = codeMcpServer(existingMcpServer);
```

**Use case:** You have an MCP server with many tools. Wrapping it with `codeMcpServer` collapses all tools into a single `code()` tool with TypeScript interfaces, dramatically reducing token usage.

**Example — Cloudflare's own MCP server:**
The Cloudflare MCP server exposes the entire Cloudflare API with just two tools and under 1,000 tokens using this approach.

## openApiMcpServer(spec)

Builds a complete MCP server from an OpenAPI specification, then wraps it with code mode.

```typescript
import { openApiMcpServer } from "@cloudflare/codemode";

const server = openApiMcpServer(myOpenApiSpec);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `spec` | `OpenAPISpec` | OpenAPI 3.x specification object or JSON |

**What it does:**
1. Parses the OpenAPI spec
2. Generates MCP tools for each endpoint
3. Wraps them with `codeMcpServer` for code-mode access
4. Exposes TypeScript type definitions derived from the spec

## Integration with AI SDKs

### Vercel AI SDK

```typescript
import { generateText } from "ai";

const { text } = await generateText({
  model,
  messages,
  tools: { codemode: createCodeTool({ tools, executor }) },
});
```

### Anthropic SDK

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
// Use codemode as a tool definition in messages API
```

### Workers AI

```typescript
const model = env.AI.binding("@cf/meta/llama-3.3-70b-instruct-fp8-fast");
// Pass codemode tool in the tools array
```

## See Also

- **[Patterns](./patterns.md)** — Real-world integration patterns
- **[Gotchas](./gotchas.md)** — Common errors
- **[Dynamic Workers API](../dynamic-workers/api.md)** — Underlying sandbox API
