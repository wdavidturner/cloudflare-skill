# @cloudflare/codemode Patterns

## Wrapping an Existing MCP Server

If you have an MCP server, wrap it to get code-mode benefits:

```typescript
import { codeMcpServer, DynamicWorkerExecutor } from "@cloudflare/codemode";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Your existing MCP server
    const myMcp = createMyMcpServer(env);

    // Wrap it — all tools become available as TypeScript APIs
    const codeMcp = codeMcpServer(myMcp);

    // Agent now writes code instead of calling tools one-by-one
    return codeMcp.handle(request);
  },
};
```

## OpenAPI to Code Mode

Turn any OpenAPI spec into a code-mode-enabled MCP server:

```typescript
import { openApiMcpServer } from "@cloudflare/codemode";

const stripeServer = openApiMcpServer(stripeOpenApiSpec);

// Agent can now write code like:
// const charges = await api.listCharges({ limit: 10 });
// const refund = await api.createRefund({ charge: charges[0].id });
```

## Multi-Service Composition

Combine multiple services into a single code-mode tool:

```typescript
const codemode = createCodeTool({
  tools: [
    ...databaseTools,    // query, execute, schema
    ...storageTools,     // read, write, list, delete
    ...emailTools,       // send, listInbox
    ...calendarTools,    // createEvent, listEvents
  ],
  executor,
});

// Agent sees all tools as TypeScript APIs and can compose freely:
// const meetings = await calendar.listEvents({ date: "2026-03-24" });
// const attendees = await db.query("SELECT email FROM users WHERE id IN (?)", ...);
// await email.send({ to: attendees, subject: `Meeting: ${meetings[0].title}` });
```

## Custom Tool Definitions

Define tools for code mode with TypeScript interfaces:

```typescript
const myTools = [
  {
    name: "database",
    description: "Query the application database",
    interface: `
      interface Database {
        query(sql: string, params?: any[]): Promise<Row[]>;
        execute(sql: string, params?: any[]): Promise<{ changes: number }>;
      }
    `,
    handler: {
      query: (sql, params) => env.DB.prepare(sql).bind(...(params || [])).all(),
      execute: (sql, params) => env.DB.prepare(sql).bind(...(params || [])).run(),
    },
  },
];
```

## Combining with Conversational Agent

Use code mode within a multi-turn agent:

```typescript
import { AgentExecutor } from "agents"; // Cloudflare Agents SDK

class MyAgent extends AgentExecutor {
  async onMessage(message: string) {
    const executor = new DynamicWorkerExecutor({
      loader: this.env.LOADER,
      globalOutbound: null,
    });

    const codemode = createCodeTool({
      tools: this.getTools(),
      executor,
    });

    const response = await generateText({
      model: this.env.AI.binding("@cf/meta/llama-3.3-70b-instruct-fp8-fast"),
      messages: [...this.history, { role: "user", content: message }],
      tools: { codemode },
    });

    this.history.push(
      { role: "user", content: message },
      { role: "assistant", content: response.text }
    );

    return response.text;
  }
}
```

## Best Practices

- **Type definitions**: Write clear, concise TypeScript interfaces — the LLM generates better code with good types
- **Batch APIs**: Expose batch operations (e.g., `getByIds(ids)` not just `getById(id)`) since the agent writes one function
- **Error messages**: Return descriptive errors from tool handlers — the agent can adapt its code
- **Scope tools**: Only include tools relevant to the task — fewer tools = fewer tokens = better code generation

## See Also

- **[API](./api.md)** — Full API reference
- **[Gotchas](./gotchas.md)** — Common errors
- **[Dynamic Workers Patterns](../dynamic-workers/patterns.md)** — Sandbox patterns
