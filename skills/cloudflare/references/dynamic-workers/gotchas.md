# Dynamic Workers Gotchas

## Common Errors

### "Sandbox Can't Access My KV/D1/R2 Binding"

**Problem:** Sandbox code references `env.MY_KV` but gets undefined
**Cause:** Dynamic Workers don't inherit harness bindings — only what you pass in `env`
**Solution:** Explicitly pass RPC stubs or wrapper objects in the `load()` options

```typescript
// ❌ Wrong — sandbox can't see harness bindings
const worker = env.LOADER.load({
  mainModule: "agent.js",
  modules: { "agent.js": code },
  // No env passed — sandbox has no bindings
});

// ✅ Right — explicitly pass what sandbox needs
const worker = env.LOADER.load({
  mainModule: "agent.js",
  modules: { "agent.js": code },
  env: {
    cache: createCacheAPI(env.KV),
    db: createDatabaseAPI(env.D1),
  },
});
```

### "Sandbox HTTP Request Blocked"

**Problem:** `fetch()` inside sandbox fails or returns 403
**Cause:** `globalOutbound: null` blocks all outbound HTTP (by design)
**Solution:** Either provide a `globalOutbound` callback or expose the needed API via RPC stubs instead

```typescript
// Sandbox can't make HTTP calls
globalOutbound: null

// Allow specific domains
globalOutbound: (req) => {
  const url = new URL(req.url);
  if (url.hostname === "api.example.com") return fetch(req);
  return new Response("Blocked", { status: 403 });
}
```

### "Module Not Found in Sandbox"

**Problem:** Sandbox import fails with module not found
**Cause:** Module filename in `import` doesn't match key in `modules` map
**Solution:** Ensure import paths exactly match the keys in the `modules` object

```typescript
// ❌ Wrong — path mismatch
env.LOADER.load({
  mainModule: "index.js",
  modules: {
    "index.js": `import { helper } from "./utils.js"; ...`,
    "src/utils.js": `export function helper() { ... }`,  // Wrong path!
  },
});

// ✅ Right — paths match
env.LOADER.load({
  mainModule: "index.js",
  modules: {
    "index.js": `import { helper } from "./utils.js"; ...`,
    "utils.js": `export function helper() { ... }`,
  },
});
```

### "Can't Import npm Packages in Sandbox"

**Problem:** Sandbox code does `import { Hono } from 'hono'` and fails
**Cause:** Dynamic Workers don't resolve npm packages — modules must be pre-bundled
**Solution:** Use `@cloudflare/worker-bundler` to bundle before loading

```typescript
import { createWorker } from "@cloudflare/worker-bundler";

const { mainModule, modules } = await createWorker({
  files: {
    "index.ts": code,
    "package.json": JSON.stringify({ dependencies: { hono: "^4.0.0" } }),
  },
});

env.LOADER.load({ mainModule, modules, compatibilityDate: "2026-03-01" });
```

### "RPC Call is Slow / High Latency"

**Problem:** Cross-isolate RPC calls feel slow
**Cause:** Each RPC call crosses the sandbox boundary with serialization overhead
**Solution:** Design coarse-grained APIs that batch operations

```typescript
// ❌ Chatty — N round-trips
for (const id of ids) {
  const item = await env.DB.getById(id);  // One RPC per item
}

// ✅ Batched — 1 round-trip
const items = await env.DB.getByIds(ids);  // Single RPC call
```

### "Sandbox Code Throws but I Can't Debug"

**Problem:** `getEntrypoint()` call throws opaque error
**Cause:** Sandbox code threw an exception
**Solution:** Wrap in try/catch and use observability

```typescript
try {
  const result = await worker.getEntrypoint().run();
  return Response.json(result);
} catch (err) {
  console.error("Sandbox error:", err.message, err.stack);
  return Response.json({ error: err.message }, { status: 500 });
}
```

### "globalOutbound Callback Doesn't See Request Body"

**Problem:** Can't read request body in globalOutbound for inspection
**Cause:** Request body is a stream — reading it consumes it
**Solution:** Clone the request before reading

```typescript
globalOutbound: async (request: Request) => {
  const clone = request.clone();
  const body = await clone.text();
  console.log("Sandbox request body:", body);
  return fetch(request);  // Forward original (unconsumed)
}
```

## Security Model

Dynamic Workers provide defense-in-depth:

1. **V8 Isolate Sandbox** — Same isolation as all Cloudflare Workers (8+ years hardened)
2. **Custom Second-Layer Sandbox** — Dynamic cordoning based on risk assessments
3. **Hardware Features** — Extended V8 sandbox with Memory Protection Keys (MPK)
4. **Spectre Defenses** — Novel mitigations developed with academic researchers
5. **Malicious Code Detection** — Automatic scanning and blocking of malicious patterns
6. **Security Patching** — V8 patches deployed within hours (faster than Chrome)

**Important:** Isolates have a more complicated attack surface than hardware VMs. For highest-risk scenarios, layer additional controls (domain allowlists, RPC-only access, no raw HTTP).

## Limits

| Limit | Value | Notes |
|-------|-------|-------|
| Isolate startup | ~1-5ms | V8 isolate cold start |
| Memory per isolate | Few MB | Much less than containers |
| Concurrent sandboxes | No global limit | Can handle 1M req/s with separate sandboxes |
| Module size | Standard Workers limits | Same as deployed Workers |
| CPU time | Standard Workers limits | Per-request CPU limits apply |
| Outbound HTTP | Controlled by globalOutbound | null = blocked |

## Language Support

- **JavaScript/TypeScript**: Optimized, fastest load and execution — recommended for AI-generated code
- **Python**: Supported via Workers Python but slower startup
- **WebAssembly**: Supported but heavier than JS for small snippets
- AI models can generate any of these — JS is preferred for token efficiency and speed

## Billing

- **$0.002 per unique Worker loaded per day** — typically negligible vs AI inference costs
- Standard Workers CPU time and invocation pricing applies
- Beta: loading charge waived
- Pricing subject to change — check docs for current rates

## See Also

- **[API](./api.md)** — Full API reference
- **[Patterns](./patterns.md)** — Workarounds and best practices
- **[Configuration](./configuration.md)** — Setup details
