# @cloudflare/shell Configuration

## Installation

```bash
npm install @cloudflare/shell
```

## Requirements

- Durable Objects with SQLite storage (for file metadata)
- R2 bucket (for file content)
- Dynamic Workers LOADER binding (for sandbox execution)
- Cloudflare Workers Paid plan

## wrangler.jsonc

```jsonc
{
  "name": "file-agent",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "unsafe": {
    "bindings": [
      { "name": "LOADER", "type": "dynamic_worker_loader" }
    ]
  },
  "durable_objects": {
    "bindings": [
      { "name": "FILE_AGENT", "class_name": "FileAgent" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["FileAgent"] }
  ],
  "r2_buckets": [
    { "binding": "FILES", "bucket_name": "workspace-files" }
  ]
}
```

## TypeScript Setup

```typescript
import { Workspace } from "@cloudflare/shell";
import { stateTools } from "@cloudflare/shell/workers";
import { DurableObject } from "cloudflare:workers";

interface Env {
  LOADER: DynamicWorkerLoader;
  FILE_AGENT: DurableObjectNamespace<FileAgent>;
  FILES: R2Bucket;
}

export class FileAgent extends DurableObject<Env> {
  private workspace: Workspace;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.workspace = new Workspace({
      sql: ctx.storage.sql,
      r2: env.FILES,
      name: () => this.ctx.id.toString(),
    });
  }
}
```

## See Also

- **[Dynamic Workers Configuration](../dynamic-workers/configuration.md)** — LOADER binding setup
- **[Durable Objects Configuration](../durable-objects/configuration.md)** — DO + SQLite setup
- **[R2 Configuration](../r2/configuration.md)** — R2 bucket setup
- **[API](./api.md)** — Library API reference
