# Dynamic Workers Patterns

## When to Use Which Pattern

| Need | Pattern | Key Detail |
|------|---------|------------|
| AI agent calling your APIs | Code Mode | Use `@cloudflare/codemode` — 81% less tokens |
| AI-generated full app | App Generation | Use `@cloudflare/worker-bundler` for npm deps |
| Agent with file operations | Workspace | Use `@cloudflare/shell` for virtual FS |
| Custom automations (CRUD, integrations) | Automation Sandbox | Pass service RPC stubs |
| Secure credential injection | Outbound Proxy | `globalOutbound` callback |
| Multi-tenant code execution | Platform Sandbox | Per-tenant LOADER.load() |

## AI Agent Sandbox (Core Pattern)

The most common pattern: an AI model generates code, your harness runs it in a sandbox with access to curated APIs.

```typescript
import { generateText } from "ai"; // Vercel AI SDK or similar

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const { prompt } = await request.json();

    // 1. AI generates code
    const { text: agentCode } = await generateText({
      model: env.AI.binding("@cf/meta/llama-3.3-70b-instruct-fp8-fast"),
      system: `You write TypeScript. Available APIs:
        env.DB.query(sql: string): Promise<Row[]>
        env.DB.execute(sql: string): Promise<void>
        Return a value from the run() function.`,
      prompt,
    });

    // 2. Execute in sandbox
    const worker = env.LOADER.load({
      compatibilityDate: "2026-03-01",
      mainModule: "agent.js",
      modules: {
        "agent.js": `export default { async run(env) { ${agentCode} } }`,
      },
      env: {
        DB: createDatabaseAPI(env.D1),
      },
      globalOutbound: null,
    });

    const result = await worker.getEntrypoint().run();
    return Response.json(result);
  },
};
```

## Code Mode (Token-Efficient Tool Use)

Instead of sequential tool calls (each requiring an LLM round-trip), the agent writes a single function that chains multiple API calls. Cuts token usage by ~81%.

```typescript
import { createCodeTool, DynamicWorkerExecutor } from "@cloudflare/codemode";

const executor = new DynamicWorkerExecutor({
  loader: env.LOADER,
  globalOutbound: null,
});

const codemode = createCodeTool({
  tools: myTools,  // Your MCP tools or custom tools
  executor,
});

// Agent writes ONE function instead of N sequential tool calls
const { text } = await generateText({
  model,
  messages,
  tools: { codemode },
});
```

See **[Codemode](../codemode/README.md)** for full reference.

## Credential Injection

Let agents call authenticated APIs without seeing secrets:

```typescript
const worker = env.LOADER.load({
  compatibilityDate: "2026-03-01",
  mainModule: "agent.js",
  modules: { "agent.js": agentCode },
  env: {},
  globalOutbound: (request: Request) => {
    const url = new URL(request.url);

    // Inject credentials per-service
    const credentials: Record<string, string> = {
      "api.stripe.com": `Bearer ${env.STRIPE_KEY}`,
      "api.github.com": `token ${env.GITHUB_TOKEN}`,
      "api.openai.com": `Bearer ${env.OPENAI_KEY}`,
    };

    const auth = credentials[url.hostname];
    if (!auth) return new Response("Blocked", { status: 403 });

    return fetch(new Request(request, {
      headers: { ...Object.fromEntries(request.headers), Authorization: auth },
    }));
  },
});
```

The sandbox code is simple — no awareness of credentials:

```typescript
// Sandbox: agent.js
export default {
  async run() {
    const charges = await fetch("https://api.stripe.com/v1/charges");
    return charges.json();
  }
};
```

## Domain Allowlist

Restrict sandbox to specific domains:

```typescript
const ALLOWED_DOMAINS = new Set([
  "api.example.com",
  "cdn.example.com",
  "data.example.com",
]);

const worker = env.LOADER.load({
  // ...
  globalOutbound: (request: Request) => {
    const url = new URL(request.url);
    if (!ALLOWED_DOMAINS.has(url.hostname)) {
      return new Response(
        JSON.stringify({ error: `Domain ${url.hostname} not allowed` }),
        { status: 403, headers: { "Content-Type": "application/json" } }
      );
    }
    return fetch(request);
  },
});
```

## Full-Stack App Generation

Use `@cloudflare/worker-bundler` to generate apps with npm dependencies:

```typescript
import { createWorker } from "@cloudflare/worker-bundler";

const worker = env.LOADER.get("user-app", async () => {
  const { mainModule, modules } = await createWorker({
    files: {
      "src/index.ts": `
        import { Hono } from 'hono';
        const app = new Hono();
        app.get('/', (c) => c.text('Hello from Hono!'));
        export default app;
      `,
      "package.json": JSON.stringify({
        dependencies: { hono: "^4.0.0" }
      }),
    },
  });

  return { mainModule, modules, compatibilityDate: "2026-01-01" };
});

return worker.getEntrypoint().fetch(request);
```

See **[Worker Bundler](../worker-bundler/README.md)** for full reference.

## Durable Object as Harness Backend

Combine Dynamic Workers with Durable Objects for stateful agent execution:

```typescript
export class AgentSession extends DurableObject<Env> {
  async executeCode(code: string): Promise<any> {
    const worker = this.env.LOADER.load({
      compatibilityDate: "2026-03-01",
      mainModule: "agent.js",
      modules: { "agent.js": code },
      env: {
        session: this.createSessionAPI(),
      },
      globalOutbound: null,
    });

    return worker.getEntrypoint().run();
  }

  private createSessionAPI() {
    return {
      getState: () => this.ctx.storage.sql.exec("SELECT * FROM state").toArray(),
      setState: (key: string, value: any) => {
        this.ctx.storage.sql.exec(
          "INSERT OR REPLACE INTO state (key, value) VALUES (?, ?)",
          key, JSON.stringify(value)
        );
      },
    };
  }
}
```

## Multi-Tenant Platform

Run customer-provided code securely:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const tenantId = getTenantFromRequest(request);
    const tenantCode = await env.KV.get(`tenant:${tenantId}:code`);

    if (!tenantCode) return new Response("Not found", { status: 404 });

    const worker = env.LOADER.load({
      compatibilityDate: "2026-03-01",
      mainModule: "index.js",
      modules: { "index.js": tenantCode },
      env: {
        // Tenant-scoped APIs only
        kv: createScopedKV(env.KV, tenantId),
        db: createScopedDB(env.DB, tenantId),
      },
      globalOutbound: null,
    });

    return worker.getEntrypoint().fetch(request);
  },
};
```

## Request Logging / Auditing

Log all sandbox HTTP requests for compliance:

```typescript
globalOutbound: async (request: Request) => {
  const start = Date.now();
  const response = await fetch(request);

  // Log to analytics
  env.ANALYTICS.writeDataPoint({
    blobs: [request.url, request.method, String(response.status)],
    doubles: [Date.now() - start],
    indexes: [tenantId],
  });

  return response;
}
```

## Best Practices

- **Security**: Always set `globalOutbound: null` unless the sandbox needs HTTP. Expose only curated RPC APIs, not raw bindings
- **Performance**: Design RPC APIs for coarse operations (batch reads, not individual lookups) to minimize cross-isolate round-trips
- **Token efficiency**: Use TypeScript interfaces instead of OpenAPI specs when prompting AI agents — significantly fewer tokens
- **Caching**: Use `env.LOADER.get(name, factory)` when the same code will be loaded multiple times to avoid re-parsing
- **Error handling**: Wrap `getEntrypoint()` calls in try/catch — sandbox code may throw

## See Also

- **[API](./api.md)** — LOADER API reference
- **[Configuration](./configuration.md)** — Setup and bindings
- **[Gotchas](./gotchas.md)** — Limits and common errors
- **[Codemode](../codemode/README.md)** — Token-efficient AI tool use
- **[Worker Bundler](../worker-bundler/README.md)** — npm dependency resolution
- **[Cloudflare Shell](../cloudflare-shell/README.md)** — Virtual filesystem for sandboxes
