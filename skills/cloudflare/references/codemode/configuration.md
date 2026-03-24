# @cloudflare/codemode Configuration

## Installation

```bash
npm install @cloudflare/codemode
```

## Requirements

- Dynamic Workers LOADER binding configured (see [Dynamic Workers Configuration](../dynamic-workers/configuration.md))
- Cloudflare Workers Paid plan

## wrangler.jsonc

Code mode requires the same LOADER binding as Dynamic Workers:

```jsonc
{
  "name": "my-agent-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "unsafe": {
    "bindings": [
      { "name": "LOADER", "type": "dynamic_worker_loader" }
    ]
  }
}
```

## TypeScript Setup

```typescript
import { createCodeTool, DynamicWorkerExecutor } from "@cloudflare/codemode";

interface Env {
  LOADER: DynamicWorkerLoader;
  AI: Ai;
  // ... other bindings
}
```

## See Also

- **[Dynamic Workers Configuration](../dynamic-workers/configuration.md)** — LOADER binding setup
- **[API](./api.md)** — Library API reference
