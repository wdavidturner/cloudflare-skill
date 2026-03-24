# @cloudflare/worker-bundler API

## createWorker(options)

Bundles source files and npm dependencies into the `{ mainModule, modules }` format for `env.LOADER.load()`.

```typescript
import { createWorker } from "@cloudflare/worker-bundler";

const { mainModule, modules } = await createWorker({
  files: {
    "src/index.ts": `
      import { Hono } from 'hono';
      const app = new Hono();
      app.get('/', (c) => c.text('Hello!'));
      export default app;
    `,
    "package.json": JSON.stringify({
      dependencies: { hono: "^4.0.0" }
    }),
  },
});
```

**Parameters:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `files` | `Record<string, string>` | Yes | Map of file path → content string |

**files format:**
- Keys are file paths (e.g., `"src/index.ts"`, `"package.json"`)
- Values are file content strings
- Must include a `package.json` if using npm dependencies
- Entry point is auto-detected or specified

**Returns:**

```typescript
{
  mainModule: string;   // Entry point filename for env.LOADER.load()
  modules: Record<string, string>;  // Bundled module map
}
```

## createApp(options)

Bundles a full-stack application: server-side Worker + client-side JavaScript + static assets, with built-in asset serving.

```typescript
import { createApp } from "@cloudflare/worker-bundler";

const { mainModule, modules } = await createApp({
  files: {
    "src/server.ts": serverCode,
    "src/client.ts": clientCode,
    "public/index.html": htmlContent,
    "public/styles.css": cssContent,
    "package.json": JSON.stringify({
      dependencies: { hono: "^4.0.0", htmx: "^2.0.0" }
    }),
  },
});
```

**Differences from createWorker:**
- Handles client-side bundling separately
- Includes static assets in the module map
- Sets up built-in asset serving routes
- Suitable for preview/playground environments

## Usage with LOADER

### One-shot loading

```typescript
const { mainModule, modules } = await createWorker({ files });

const worker = env.LOADER.load({
  mainModule,
  modules,
  compatibilityDate: "2026-03-01",
});
```

### Cached loading (recommended for repeated use)

```typescript
const worker = env.LOADER.get("app-name", async () => {
  const { mainModule, modules } = await createWorker({ files });
  return { mainModule, modules, compatibilityDate: "2026-03-01" };
});
```

The factory only runs once — subsequent calls return the cached worker.

## See Also

- **[Patterns](./patterns.md)** — Full-stack apps, AI-generated apps
- **[Gotchas](./gotchas.md)** — Bundle limits, dependency issues
- **[Dynamic Workers API](../dynamic-workers/api.md)** — LOADER.load() and LOADER.get()
