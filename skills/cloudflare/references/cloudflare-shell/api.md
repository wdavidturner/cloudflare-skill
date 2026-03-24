# @cloudflare/shell API

## Workspace

The core class that provides a virtual filesystem backed by SQLite and R2.

### Constructor

```typescript
import { Workspace } from "@cloudflare/shell";

const workspace = new Workspace({
  sql: this.ctx.storage.sql,     // Durable Object SQLite
  r2: this.env.MY_BUCKET,        // R2 bucket for file content
  name: () => this.name,         // Workspace identifier
});
```

**Parameters:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `sql` | `SqlStorage` | Yes | Durable Object SQLite storage |
| `r2` | `R2Bucket` | Yes | R2 bucket for file content |
| `name` | `() => string` | Yes | Function returning workspace name |

## stateTools(workspace)

Creates tool providers that expose the workspace to Dynamic Worker sandboxes as a `state` object.

```typescript
import { stateTools } from "@cloudflare/shell/workers";

const tools = stateTools(workspace);
// Pass to executor.execute() as a provider
```

## File Operations (available as `state.*` in sandbox)

### state.readFile(path)

Read a file's content.

```typescript
const content = await state.readFile("/src/app.ts");
```

### state.writeFile(path, content)

Write content to a file (creates or overwrites).

```typescript
await state.writeFile("/src/app.ts", "export default {}");
```

### state.searchFiles(glob, query)

Search for files matching a glob pattern with optional content matching.

```typescript
// Find all TypeScript files containing "TODO"
const hits = await state.searchFiles("src/**/*.ts", "TODO");
// Returns: [{ path: "/src/app.ts", matches: [...] }, ...]
```

### state.glob(pattern)

List files matching a glob pattern.

```typescript
const files = await state.glob("src/**/*.ts");
// Returns: ["/src/app.ts", "/src/utils.ts", ...]
```

### state.diff(path, newContent)

Get a diff between current file content and proposed new content.

```typescript
const diff = await state.diff("/src/app.ts", newAppCode);
```

## Edit Planning

Two-phase editing: plan changes, then apply atomically.

### state.planEdits(operations)

Create an edit plan from an array of operations. Does not modify files — returns a plan for review.

```typescript
const plan = await state.planEdits([
  {
    kind: "replace",
    path: "/src/app.ts",
    search: "oldValue",
    replacement: "newValue",
  },
  {
    kind: "write",
    path: "/src/new-file.ts",
    content: "export const x = 1;",
  },
  {
    kind: "writeJson",
    path: "/config.json",
    value: { version: 2 },
  },
  {
    kind: "delete",
    path: "/src/deprecated.ts",
  },
]);
```

**Edit operation kinds:**

| Kind | Fields | Description |
|------|--------|-------------|
| `replace` | `path`, `search`, `replacement` | Find and replace text in a file |
| `write` | `path`, `content` | Write/overwrite a file |
| `writeJson` | `path`, `value` | Write a JSON file |
| `delete` | `path` | Delete a file |

### state.applyEditPlan(plan)

Apply a previously created edit plan. Atomic — all operations succeed or all roll back.

```typescript
const result = await state.applyEditPlan(plan);
// result contains: applied operations, any errors
```

**Transactional guarantees:**
- All operations apply atomically
- If any operation fails, all changes roll back
- Safe for multi-file refactoring

## JSON Operations

### state.readJson(path)

Read and parse a JSON file.

```typescript
const config = await state.readJson("/config.json");
```

### state.writeJson(path, value)

Write a JavaScript object as formatted JSON.

```typescript
await state.writeJson("/config.json", { version: 2, debug: false });
```

### state.updateJson(path, updater)

Read, transform, and write a JSON file atomically.

```typescript
await state.updateJson("/package.json", (pkg) => {
  pkg.version = "2.0.0";
  return pkg;
});
```

## Archive Operations

Create and extract archives from workspace files.

```typescript
// Create archive from workspace files
const archive = await state.createArchive("src/**/*.ts");

// Extract archive into workspace
await state.extractArchive(archiveBuffer, "/extracted/");
```

## See Also

- **[Patterns](./patterns.md)** — Real-world usage patterns
- **[Gotchas](./gotchas.md)** — Storage limits, transactional caveats
- **[Dynamic Workers API](../dynamic-workers/api.md)** — Sandbox execution
