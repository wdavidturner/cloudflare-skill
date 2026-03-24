# @cloudflare/worker-bundler

Expert guidance for using `@cloudflare/worker-bundler` — pre-bundles modules with npm dependencies for Dynamic Workers.

## Reading Order

1. **First time?** Read this overview + Quick Start
2. **Building features?** See [Patterns](./patterns.md)
3. **API details?** See [API](./api.md)
4. **Debugging issues?** Check [Gotchas](./gotchas.md)

## Overview

Dynamic Workers require all modules to be pre-bundled — they can't resolve npm packages at runtime. `@cloudflare/worker-bundler` solves this by using esbuild to resolve dependencies and produce the `{ mainModule, modules }` format that `env.LOADER.load()` expects.

### Key Capabilities

- **npm dependency resolution**: Resolves `package.json` dependencies via esbuild
- **TypeScript support**: Bundles `.ts` files directly
- **Full-stack apps**: `createApp()` bundles server + client JS + static assets together
- **Built-in asset serving**: Static assets served automatically
- **Caching**: Use with `env.LOADER.get(name, factory)` to avoid re-bundling

## Quick Start

```typescript
import { createWorker } from "@cloudflare/worker-bundler";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const worker = env.LOADER.get("my-app", async () => {
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
  },
};
```

## Decision Trees

### When to use worker-bundler?

```
Dynamic code needs npm packages?
├─ Yes, full app with dependencies → createWorker() or createApp()
├─ Yes, but just a few imports → createWorker() with package.json
├─ No, self-contained code → Direct env.LOADER.load() (no bundler needed)
└─ Need client-side JS + static assets → createApp()
```

## Resources

**Blog**: https://blog.cloudflare.com/dynamic-workers/
**npm**: `@cloudflare/worker-bundler`

## In This Reference

- **[Configuration](./configuration.md)** — Installation and setup
- **[API](./api.md)** — createWorker, createApp, file format
- **[Patterns](./patterns.md)** — Full-stack apps, cached bundling, AI-generated apps
- **[Gotchas](./gotchas.md)** — Bundle size limits, dependency issues

## See Also

- **[Dynamic Workers](../dynamic-workers/README.md)** — The runtime that loads bundled code
- **[Codemode](../codemode/README.md)** — Code mode (doesn't need bundler for simple functions)
- **[Static Assets](../static-assets/README.md)** — Static asset serving
