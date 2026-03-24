# @cloudflare/worker-bundler Gotchas

## Common Errors

### "Dependency Resolution Failed"

**Problem:** `createWorker()` fails trying to resolve a package
**Cause:** Package not in the `dependencies` field, or private/unavailable package
**Solution:** Ensure all imported packages are listed in `package.json` dependencies

```typescript
// ❌ Missing dependency
files: {
  "index.ts": `import { z } from 'zod'; ...`,
  "package.json": JSON.stringify({ dependencies: {} }),  // zod missing!
}

// ✅ Dependencies declared
files: {
  "index.ts": `import { z } from 'zod'; ...`,
  "package.json": JSON.stringify({ dependencies: { zod: "^3.0.0" } }),
}
```

### "Bundle Too Large"

**Problem:** Bundled output exceeds Worker size limits
**Cause:** Large dependency tree (e.g., pulling in all of AWS SDK)
**Solution:** Use smaller packages, tree-shake, import specific submodules

### "createWorker vs createApp Confusion"

**Problem:** Static assets not served, or client JS not bundled
**Cause:** Using `createWorker` when you need full-stack bundling
**Solution:** Use `createApp` for apps with client-side JS and static assets; `createWorker` for server-only Workers

### "Bundling is Slow"

**Problem:** `createWorker()` takes too long
**Cause:** esbuild resolution + large dependency trees
**Solution:** Cache with `env.LOADER.get(name, factory)` — the factory only runs once

### "Entry Point Not Found"

**Problem:** Bundle succeeds but sandbox can't find the main module
**Cause:** Entry point auto-detection failed or file structure is ambiguous
**Solution:** Ensure you have a clear entry point (e.g., `src/index.ts` or `index.ts`)

## See Also

- **[API](./api.md)** — API reference
- **[Dynamic Workers Gotchas](../dynamic-workers/gotchas.md)** — Sandbox-level issues
