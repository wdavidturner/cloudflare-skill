# Cloudflare Dynamic Workers

Expert guidance for building AI agent sandboxes and dynamic code execution with Cloudflare Dynamic Workers.

## Reading Order

1. **First time?** Read this overview + Quick Start
2. **Setting up?** See [Configuration](./configuration.md)
3. **Building features?** Use decision trees below → [Patterns](./patterns.md)
4. **Debugging issues?** Check [Gotchas](./gotchas.md)
5. **Deep dive?** [API](./api.md)

## Overview

Dynamic Workers enable secure execution of AI-generated or user-provided code in V8 isolates at the edge. Instead of deploying a Worker via wrangler, you load code at runtime from a parent (harness) Worker.

- **100x faster than containers**: V8 isolates start in milliseconds, use a few MB of memory
- **Global scale**: Available in every Cloudflare location (hundreds of PoPs)
- **No concurrency limits**: Can handle a million requests/second, each loading a separate sandbox
- **Defense-in-depth security**: V8 sandboxing + custom sandbox layer + hardware features (MPK) + Spectre defenses + malicious code scanning
- **RPC bridge**: Automatic Cap'n Proto RPC between sandbox and harness code

## Key Concepts

### Architecture: Harness + Sandbox

Dynamic Workers follow a two-layer pattern:

1. **Harness Worker** — Your deployed Worker that controls sandbox creation, passes bindings, and manages outbound HTTP
2. **Sandbox (Dynamic Worker)** — The dynamically-loaded code that runs in an isolated V8 isolate with only the APIs you explicitly provide

The harness calls `env.LOADER.load()` to create a sandbox, passing it code modules and a curated set of env bindings via RPC stubs.

### What Makes This Different from Regular Workers

| Aspect | Regular Workers | Dynamic Workers |
|--------|----------------|-----------------|
| Deployment | `wrangler deploy` | Loaded at runtime via `env.LOADER.load()` |
| Code source | Your codebase | AI-generated, user-provided, or dynamic |
| Bindings | Configured in wrangler.jsonc | Passed explicitly via `env` option |
| HTTP access | Full outbound | Controlled via `globalOutbound` callback |
| Lifecycle | Long-lived deployment | Per-request or cached sandbox |
| Use case | Your application | Untrusted/dynamic code execution |

### TypeScript Interfaces for Agents

Instead of verbose OpenAPI specs, expose typed interfaces to AI agents — significantly less tokens:

```typescript
interface ChatRoom {
  getHistory(limit: number): Promise<Message[]>;
  subscribe(callback: (msg: Message) => void): Promise<Disposable>;
  post(text: string): Promise<void>;
}

type Message = {
  author: string;
  time: Date;
  text: string;
}
```

The Workers Runtime automatically sets up Cap'n Proto RPC bridges between the sandbox and your harness code, so agents call methods like `env.CHAT_ROOM.getHistory(1000)` directly.

## Quick Start

### Harness Worker (your deployed Worker)

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const agentCode = `
      export default {
        async run(env) {
          const history = await env.CHAT_ROOM.getHistory(10);
          return history.map(m => m.text).join("\\n");
        }
      }
    `;

    const worker = env.LOADER.load({
      compatibilityDate: "2026-03-01",
      mainModule: "agent.js",
      modules: { "agent.js": agentCode },
      env: { CHAT_ROOM: chatRoomRpcStub },
      globalOutbound: null, // Block all outbound HTTP
    });

    const result = await worker.getEntrypoint().run();
    return new Response(result);
  }
};
```

### Sandbox Code (AI-generated / dynamic)

```typescript
export default {
  async run(env) {
    const history = await env.CHAT_ROOM.getHistory(1000);
    return history.filter(msg => msg.author === "alice");
  }
}
```

## Decision Trees

### Which dynamic execution approach?

```
Need to run dynamic code?
├─ AI agent writing code against your APIs → Dynamic Workers + codemode/
├─ AI agent generating full apps with npm deps → Dynamic Workers + worker-bundler/
├─ Agent needs filesystem operations → Dynamic Workers + cloudflare-shell/
├─ Multi-tenant platform (customers deploy Workers) → workers-for-platforms/
└─ Static sandboxing (no dynamic code) → Regular Workers
```

### How should the sandbox access services?

```
Sandbox needs to call external services?
├─ Block all HTTP → globalOutbound: null
├─ Inspect/rewrite requests → globalOutbound: callback function
├─ Inject auth credentials (agent never sees secrets) → globalOutbound with credential injection
└─ Expose specific APIs only → Pass RPC stubs via env option
```

## Pricing

- **$0.002 per unique Worker loaded per day** (plus standard CPU time and invocation pricing)
- Charge waived during open beta
- Requires Workers Paid plan

## Essential Commands

```bash
# Use the Dynamic Workers Starter
npm create cloudflare@latest -- --template=dynamic-workers-starter

# Or the Playground (code editor + bundling + execution)
npm create cloudflare@latest -- --template=dynamic-workers-playground
```

## Resources

**Blog**: https://blog.cloudflare.com/dynamic-workers/
**Docs**: https://developers.cloudflare.com/workers/runtime-apis/dynamic-workers/

## In This Reference

- **[Configuration](./configuration.md)** — wrangler.jsonc setup, LOADER binding, compatibility settings
- **[API](./api.md)** — Loader API, load() options, getEntrypoint(), globalOutbound
- **[Patterns](./patterns.md)** — AI agent sandboxes, code mode, credential injection, multi-module
- **[Gotchas](./gotchas.md)** — Security model, limits, common errors

## See Also

- **[Codemode](../codemode/README.md)** — `@cloudflare/codemode` for AI tool-use code generation
- **[Worker Bundler](../worker-bundler/README.md)** — `@cloudflare/worker-bundler` for npm dependency resolution
- **[Cloudflare Shell](../cloudflare-shell/README.md)** — `@cloudflare/shell` for virtual filesystem in sandboxes
- **[Workers for Platforms](../workers-for-platforms/README.md)** — Multi-tenant Worker deployment (static, not dynamic)
- **[Workers](../workers/README.md)** — Core Workers runtime
- **[Durable Objects](../durable-objects/README.md)** — Stateful coordination (often used as harness backend)
