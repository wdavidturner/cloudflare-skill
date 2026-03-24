# @cloudflare/worker-bundler Configuration

## Installation

```bash
npm install @cloudflare/worker-bundler
```

## Requirements

- Dynamic Workers LOADER binding configured (see [Dynamic Workers Configuration](../dynamic-workers/configuration.md))
- Cloudflare Workers Paid plan

## wrangler.jsonc

Same LOADER binding as Dynamic Workers:

```jsonc
{
  "name": "my-app-generator",
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
import { createWorker, createApp } from "@cloudflare/worker-bundler";

interface Env {
  LOADER: DynamicWorkerLoader;
}
```

## See Also

- **[Dynamic Workers Configuration](../dynamic-workers/configuration.md)** — LOADER binding setup
- **[API](./api.md)** — Library API reference
