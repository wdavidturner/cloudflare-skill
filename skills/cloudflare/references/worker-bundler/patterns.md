# @cloudflare/worker-bundler Patterns

## AI-Generated Full-Stack App

LLM generates a complete Hono app with dependencies:

```typescript
const appCode = await generateAppCode(prompt); // AI generates code

const worker = env.LOADER.get(`app-${appId}`, async () => {
  const { mainModule, modules } = await createWorker({
    files: {
      "src/index.ts": appCode,
      "package.json": JSON.stringify({
        dependencies: {
          hono: "^4.0.0",
          zod: "^3.0.0",
        },
      }),
    },
  });
  return { mainModule, modules, compatibilityDate: "2026-03-01" };
});

return worker.getEntrypoint().fetch(request);
```

## App Preview / Playground

Build a code editor playground where users write code and see live previews:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === "POST") {
      const { files } = await request.json();

      const { mainModule, modules } = await createApp({ files });
      const worker = env.LOADER.load({
        mainModule,
        modules,
        compatibilityDate: "2026-03-01",
      });

      // Return the preview response
      return worker.getEntrypoint().fetch(
        new Request("https://preview.local/", { method: "GET" })
      );
    }

    return new Response("POST files to preview", { status: 405 });
  },
};
```

## Cached Bundling with Versioning

Cache bundles by content hash to avoid re-bundling unchanged code:

```typescript
async function getOrCreateWorker(env: Env, code: string) {
  const hash = await crypto.subtle.digest(
    "SHA-256",
    new TextEncoder().encode(code)
  );
  const cacheKey = Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, "0"))
    .join("");

  return env.LOADER.get(`worker-${cacheKey}`, async () => {
    const { mainModule, modules } = await createWorker({
      files: {
        "index.ts": code,
        "package.json": JSON.stringify({ dependencies: {} }),
      },
    });
    return { mainModule, modules, compatibilityDate: "2026-03-01" };
  });
}
```

## Multi-File App from User Input

Accept a file tree from the client and bundle it:

```typescript
// Client sends: { "src/index.ts": "...", "src/utils.ts": "...", "package.json": "..." }
const files = await request.json();

const { mainModule, modules } = await createApp({ files });
const worker = env.LOADER.load({
  mainModule,
  modules,
  compatibilityDate: "2026-03-01",
  globalOutbound: null,
});

return worker.getEntrypoint().fetch(request);
```

## Best Practices

- **Cache bundles**: Use `env.LOADER.get(name, factory)` to avoid re-bundling the same code
- **Pin dependencies**: Use exact versions in package.json for reproducibility
- **Minimize deps**: Fewer dependencies = faster bundling = smaller bundles
- **Use createApp for full-stack**: When you need client JS + static assets, use `createApp` over `createWorker`

## See Also

- **[API](./api.md)** — createWorker, createApp API reference
- **[Gotchas](./gotchas.md)** — Bundle size limits
- **[Dynamic Workers Patterns](../dynamic-workers/patterns.md)** — Broader sandbox patterns
