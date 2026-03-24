# @cloudflare/shell

Expert guidance for using `@cloudflare/shell` — a virtual filesystem for Dynamic Workers, backed by SQLite and R2.

## Reading Order

1. **First time?** Read this overview + Quick Start
2. **Building features?** See [Patterns](./patterns.md)
3. **API details?** See [API](./api.md)
4. **Debugging issues?** Check [Gotchas](./gotchas.md)

## Overview

`@cloudflare/shell` provides a Workspace abstraction that gives Dynamic Worker sandboxes structured file operations — read, write, search, replace, diff, glob, JSON manipulation, and archiving. Storage is backed by Durable Object SQLite and R2 with transactional writes and automatic rollback on failure.

### Key Capabilities

- **File operations**: read, write, search, replace, diff, glob
- **JSON operations**: query and update JSON files
- **Archive operations**: create and extract archives
- **Transactional writes**: Atomic operations with automatic rollback on failure
- **Dual storage**: SQLite (Durable Objects) for metadata + R2 for file content
- **Edit planning**: `planEdits()` + `applyEditPlan()` for safe multi-file changes
- **Search**: `searchFiles()` with glob patterns and content matching

## Quick Start

```typescript
import { Workspace } from "@cloudflare/shell";
import { stateTools } from "@cloudflare/shell/workers";
import { DynamicWorkerExecutor } from "@cloudflare/codemode";

export class FileAgent extends DurableObject<Env> {
  async executeEdit(code: string) {
    const workspace = new Workspace({
      sql: this.ctx.storage.sql,
      r2: this.env.MY_BUCKET,
      name: () => this.name,
    });

    const executor = new DynamicWorkerExecutor({
      loader: this.env.LOADER,
    });

    return executor.execute(code, [resolveProvider(stateTools(workspace))]);
  }
}
```

### Sandbox code using the workspace

```typescript
// This code runs inside the Dynamic Worker sandbox
async () => {
  // Search for files
  const hits = await state.searchFiles("src/**/*.ts", "TODO");

  // Plan edits (preview before applying)
  const plan = await state.planEdits([
    {
      kind: "replace",
      path: "/src/app.ts",
      search: "TODO: implement",
      replacement: "// Implemented",
    },
    {
      kind: "writeJson",
      path: "/src/config.json",
      value: { version: 2, updated: true },
    },
  ]);

  // Apply atomically (rolls back on failure)
  return await state.applyEditPlan(plan);
}
```

## Decision Trees

### When to use @cloudflare/shell?

```
Agent needs file operations?
├─ Read/write/search files in a workspace → @cloudflare/shell
├─ Just executing code against APIs → @cloudflare/codemode (no shell needed)
├─ Full app with npm deps → @cloudflare/worker-bundler (no shell needed)
└─ Simple key-value storage → KV or Durable Object storage directly
```

### Which storage backend?

```
Workspace storage?
├─ Durable Object with SQLite → sql: this.ctx.storage.sql (recommended)
├─ R2 for large files → r2: this.env.MY_BUCKET
└─ Both → SQLite for metadata + R2 for content (default)
```

## Resources

**Blog**: https://blog.cloudflare.com/dynamic-workers/
**npm**: `@cloudflare/shell`

## In This Reference

- **[Configuration](./configuration.md)** — Installation, storage backend setup
- **[API](./api.md)** — Workspace, stateTools, file operations, edit planning
- **[Patterns](./patterns.md)** — Code editing agents, file search, transactional edits
- **[Gotchas](./gotchas.md)** — Storage limits, transactional caveats

## See Also

- **[Dynamic Workers](../dynamic-workers/README.md)** — The sandbox runtime
- **[Codemode](../codemode/README.md)** — Code mode tool for AI agents
- **[Durable Objects](../durable-objects/README.md)** — SQLite storage backend
- **[R2](../r2/README.md)** — Object storage backend
