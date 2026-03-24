# Dynamic Workers API

## LOADER Binding

The `env.LOADER` binding is the entry point for creating Dynamic Worker sandboxes.

### `env.LOADER.load(options)`

Creates and returns a Dynamic Worker sandbox.

```typescript
const worker = env.LOADER.load({
  compatibilityDate: "2026-03-01",
  mainModule: "agent.js",
  modules: { "agent.js": code },
  env: { /* RPC stubs and bindings */ },
  globalOutbound: null,
});
```

**Parameters:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `compatibilityDate` | `string` | Yes | Runtime version date (e.g., `"2026-03-01"`) |
| `mainModule` | `string` | Yes | Entry point filename (must match key in `modules`) |
| `modules` | `Record<string, string>` | Yes | Map of filename → code string |
| `env` | `Record<string, any>` | No | RPC stubs and bindings exposed to sandbox |
| `globalOutbound` | `null \| Function` | No | Controls outbound HTTP from sandbox |

**Returns:** A Worker object with `getEntrypoint()` method.

### `env.LOADER.get(name, factory)`

Creates or retrieves a cached Dynamic Worker by name.

```typescript
const worker = env.LOADER.get("my-worker", async () => {
  return {
    mainModule: "index.js",
    modules: { "index.js": code },
    compatibilityDate: "2026-03-01",
  };
});
```

The factory function runs only on first access — subsequent calls return the cached worker. Useful with `@cloudflare/worker-bundler` to avoid re-bundling.

## Worker Object

### `worker.getEntrypoint(name?)`

Returns an RPC stub for the sandbox's default export (or named export).

```typescript
// Default entrypoint
const entry = worker.getEntrypoint();
await entry.myMethod(arg1, arg2);

// Named entrypoint
const named = worker.getEntrypoint("namedExport");
```

All method calls use Cap'n Proto RPC — serialization is automatic. Methods can return:
- Primitives (string, number, boolean)
- Objects and arrays (JSON-serializable)
- Promises
- ReadableStream

### `worker.getEntrypoint().fetch(request)`

Call the sandbox's `fetch` handler directly:

```typescript
const response = await worker.getEntrypoint().fetch(request);
```

## globalOutbound

Controls how the sandbox can make outbound HTTP requests.

### Block All HTTP

```typescript
globalOutbound: null  // Sandbox cannot make any HTTP requests
```

### Inspect and Rewrite Requests

```typescript
globalOutbound: (request: Request) => {
  // Log what the sandbox is requesting
  console.log(`Sandbox requesting: ${request.url}`);

  // Allow only specific domains
  const url = new URL(request.url);
  if (!["api.example.com", "cdn.example.com"].includes(url.hostname)) {
    return new Response("Forbidden", { status: 403 });
  }

  // Forward with modifications
  return fetch(request);
}
```

### Credential Injection

Inject auth credentials without exposing them to the sandbox:

```typescript
globalOutbound: (request: Request) => {
  const url = new URL(request.url);

  if (url.hostname === "api.stripe.com") {
    const authed = new Request(request, {
      headers: {
        ...Object.fromEntries(request.headers),
        "Authorization": `Bearer ${env.STRIPE_SECRET}`,
      },
    });
    return fetch(authed);
  }

  return new Response("Blocked", { status: 403 });
}
```

The sandbox code simply calls `fetch("https://api.stripe.com/v1/charges")` — it never sees the secret key.

### Return Direct Responses

```typescript
globalOutbound: (request: Request) => {
  // Mock/intercept specific endpoints
  if (request.url === "https://api.example.com/config") {
    return new Response(JSON.stringify({ version: 2 }), {
      headers: { "Content-Type": "application/json" },
    });
  }
  return fetch(request);
}
```

## Sandbox Module Format

Sandbox code uses standard ES module format with a default export:

```typescript
// agent.js — loaded as mainModule
export default {
  // Custom RPC methods (called via getEntrypoint())
  async myAgent(param, env, ctx) {
    const data = await env.MY_SERVICE.getData();
    return { result: data };
  },

  // Standard fetch handler (optional)
  async fetch(request, env, ctx) {
    return new Response("Hello from sandbox");
  },
};
```

### Multi-Module Sandboxes

Pass multiple modules and use standard imports:

```typescript
const worker = env.LOADER.load({
  compatibilityDate: "2026-03-01",
  mainModule: "index.js",
  modules: {
    "index.js": `
      import { helper } from "./utils.js";
      export default {
        async run(env) { return helper(await env.API.getData()); }
      }
    `,
    "utils.js": `
      export function helper(data) { return data.map(d => d.name); }
    `,
  },
  env: { API: apiStub },
  globalOutbound: null,
});
```

## RPC Bridge

The Workers Runtime automatically creates Cap'n Proto RPC bridges between sandbox and harness. When you pass an object in the `env` option, each method becomes callable across the isolate boundary.

```typescript
// Harness side — expose methods to sandbox
class ChatRoomAPI {
  async getHistory(limit: number): Promise<Message[]> { /* ... */ }
  async post(text: string): Promise<void> { /* ... */ }
}

const worker = env.LOADER.load({
  // ...
  env: { CHAT_ROOM: new ChatRoomAPI() },
});

// Sandbox side — call methods directly
export default {
  async run(env) {
    const msgs = await env.CHAT_ROOM.getHistory(100);
    await env.CHAT_ROOM.post("Hello!");
  }
};
```

**RPC design principles:**
- Design for coarse operations to minimize round-trips
- Each RPC call crosses the isolate boundary
- Arguments and return values are serialized via Cap'n Proto
- Prefer batch operations (e.g., `getHistory(100)`) over chatty calls

## Types

```typescript
interface Env {
  LOADER: DynamicWorkerLoader;
}

interface DynamicWorkerLoader {
  load(options: LoadOptions): DynamicWorker;
  get(name: string, factory: () => Promise<LoadOptions>): DynamicWorker;
}

interface LoadOptions {
  compatibilityDate: string;
  mainModule: string;
  modules: Record<string, string>;
  env?: Record<string, any>;
  globalOutbound?: null | ((request: Request) => Response | Promise<Response>);
}

interface DynamicWorker {
  getEntrypoint(name?: string): DynamicWorkerEntrypoint;
}
```

## See Also

- **[Configuration](./configuration.md)** — LOADER binding setup
- **[Patterns](./patterns.md)** — Real-world usage patterns
- **[Gotchas](./gotchas.md)** — Limits and common errors
