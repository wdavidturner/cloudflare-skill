# @cloudflare/shell Patterns

## Code Editing Agent

An AI agent that can search, read, and edit files in a workspace:

```typescript
export class CodeAgent extends DurableObject<Env> {
  private workspace: Workspace;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.workspace = new Workspace({
      sql: ctx.storage.sql,
      r2: env.FILES,
      name: () => this.ctx.id.toString(),
    });
  }

  async runAgent(prompt: string) {
    const executor = new DynamicWorkerExecutor({ loader: this.env.LOADER });

    // AI generates edit code
    const { text: editCode } = await generateText({
      model: this.env.AI.binding("@cf/meta/llama-3.3-70b-instruct-fp8-fast"),
      system: `You write TypeScript that uses these state APIs:
        state.readFile(path): Promise<string>
        state.writeFile(path, content): Promise<void>
        state.searchFiles(glob, query): Promise<SearchHit[]>
        state.planEdits(ops): Promise<EditPlan>
        state.applyEditPlan(plan): Promise<Result>
        Return the result of your operations.`,
      prompt,
    });

    return executor.execute(editCode, [resolveProvider(stateTools(this.workspace))]);
  }
}
```

## Safe Multi-File Refactoring

Use plan + apply for atomic refactoring across files:

```typescript
async () => {
  // Find all files importing the old module
  const hits = await state.searchFiles("src/**/*.ts", "from './old-module'");

  // Plan all replacements
  const plan = await state.planEdits(
    hits.map(hit => ({
      kind: "replace",
      path: hit.path,
      search: "from './old-module'",
      replacement: "from './new-module'",
    }))
  );

  // Apply atomically — all succeed or all rollback
  return await state.applyEditPlan(plan);
}
```

## File Explorer Agent

Give an agent the ability to explore and understand a codebase:

```typescript
async () => {
  // List all source files
  const files = await state.glob("src/**/*.{ts,tsx}");

  // Read key files
  const entrypoint = await state.readFile("/src/index.ts");
  const config = await state.readJson("/tsconfig.json");

  // Search for patterns
  const apiRoutes = await state.searchFiles("src/**/*.ts", "app.get\\(|app.post\\(");

  return {
    fileCount: files.length,
    entrypoint,
    config,
    apiRoutes: apiRoutes.map(h => ({ path: h.path, matches: h.matches })),
  };
}
```

## Workspace Initialization

Seed a workspace with initial project files:

```typescript
async initWorkspace(template: string) {
  const templates: Record<string, Record<string, string>> = {
    "hono-api": {
      "/src/index.ts": `import { Hono } from 'hono';\nconst app = new Hono();\nexport default app;`,
      "/package.json": JSON.stringify({ dependencies: { hono: "^4.0.0" } }, null, 2),
      "/tsconfig.json": JSON.stringify({ compilerOptions: { target: "ES2022" } }, null, 2),
    },
  };

  const files = templates[template];
  for (const [path, content] of Object.entries(files)) {
    await this.workspace.writeFile(path, content);
  }
}
```

## JSON Config Management

Agents reading and updating configuration:

```typescript
async () => {
  // Read current config
  const pkg = await state.readJson("/package.json");

  // Update version
  await state.updateJson("/package.json", (p) => {
    p.version = "2.0.0";
    return p;
  });

  // Add a new config file
  await state.writeJson("/src/config.json", {
    apiUrl: "https://api.example.com",
    debug: false,
    features: ["auth", "billing"],
  });

  return { previousVersion: pkg.version, newVersion: "2.0.0" };
}
```

## Best Practices

- **Plan before apply**: Always use `planEdits()` + `applyEditPlan()` for multi-file changes — atomic rollback protects against partial failures
- **Glob before read**: Use `searchFiles()` or `glob()` to find relevant files before reading — avoids reading everything
- **Scope workspaces**: One Durable Object per workspace (project/user/session) — natural isolation boundary
- **Use R2 for large files**: SQLite handles metadata; R2 handles large file content efficiently

## See Also

- **[API](./api.md)** — Full API reference
- **[Gotchas](./gotchas.md)** — Storage limits, transaction caveats
- **[Durable Objects Patterns](../durable-objects/patterns.md)** — DO patterns for the storage backend
