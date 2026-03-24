# Dynamic Workers Configuration

## Requirements

- **Cloudflare Workers Paid plan** (required)
- **Status**: Open beta (available to all paid Workers users)

## Basic Setup

### wrangler.jsonc

```jsonc
{
  "name": "my-harness-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "observability": { "enabled": true },
  "unsafe": {
    "bindings": [
      {
        "name": "LOADER",
        "type": "dynamic_worker_loader"
      }
    ]
  }
}
```

The `LOADER` binding is declared under `unsafe.bindings` with type `dynamic_worker_loader`.

## TypeScript Types

```typescript
interface Env {
  LOADER: DynamicWorkerLoader;
  // Your other bindings (KV, D1, R2, DO, etc.)
}
```

Generate types with:

```bash
npx wrangler types
```

## Compatibility Date

The `compatibilityDate` passed to `env.LOADER.load()` controls the runtime version for the **sandbox**, independent of the harness Worker's own compatibility_date:

```typescript
env.LOADER.load({
  compatibilityDate: "2026-03-01",  // Sandbox runtime version
  // ...
});
```

This means:
- **Harness Worker**: Uses `compatibility_date` from wrangler.jsonc
- **Sandbox**: Uses `compatibilityDate` from the `load()` call
- They can differ — useful for pinning sandbox behavior

## Combining with Other Bindings

Dynamic Workers are typically used alongside other Cloudflare products. The harness Worker holds the bindings and selectively exposes them to sandboxes via RPC stubs:

```jsonc
{
  "name": "ai-agent-harness",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "unsafe": {
    "bindings": [
      { "name": "LOADER", "type": "dynamic_worker_loader" }
    ]
  },
  "kv_namespaces": [
    { "binding": "CACHE", "id": "abc123" }
  ],
  "d1_databases": [
    { "binding": "DB", "database_name": "mydb", "database_id": "def456" }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "CHAT_ROOMS", "class_name": "ChatRoom" }
    ]
  },
  "r2_buckets": [
    { "binding": "STORAGE", "bucket_name": "my-bucket" }
  ]
}
```

Then in your harness code, wrap and expose only what the sandbox needs:

```typescript
const worker = env.LOADER.load({
  compatibilityDate: "2026-03-01",
  mainModule: "agent.js",
  modules: { "agent.js": agentCode },
  env: {
    // Expose a curated API, not raw bindings
    cache: createCacheAPI(env.CACHE),
    db: createDatabaseAPI(env.DB),
    chat: chatRoomStub,
  },
  globalOutbound: null,
});
```

## Project Starters

```bash
# Hello world example
npm create cloudflare@latest -- --template=dynamic-workers-starter

# Code editor playground with bundling and execution
npm create cloudflare@latest -- --template=dynamic-workers-playground
```

## Development

```bash
npx wrangler dev           # Local development
npx wrangler deploy        # Deploy harness Worker
```

## See Also

- **[API](./api.md)** — LOADER binding API reference
- **[Patterns](./patterns.md)** — Common integration patterns
- **[Gotchas](./gotchas.md)** — Common configuration errors
