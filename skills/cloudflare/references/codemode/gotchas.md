# @cloudflare/codemode Gotchas

## Common Errors

### "Agent Generates Invalid Code"

**Problem:** AI model writes code that doesn't parse or run
**Cause:** Model hallucinating API methods that don't exist, or syntax errors
**Solution:** Ensure TypeScript interfaces are clear and complete. Use a capable model (Llama 3.3 70B+ or Claude/GPT-4 class)

### "Tool Not Available in Sandbox"

**Problem:** Agent code references a tool that wasn't provided
**Cause:** Tool not included in the `tools` array passed to `createCodeTool()`
**Solution:** Verify all tools the agent might need are included

### "Code Mode Uses More Tokens Than Expected"

**Problem:** Token usage higher than standard tool calls
**Cause:** Too many tools exposed — all type definitions are sent as context
**Solution:** Scope tools to what's relevant for the current task. Don't dump every tool into every code-mode call

### "Sandbox Execution Timeout"

**Problem:** Code mode function times out
**Cause:** Agent wrote an infinite loop or very expensive operation
**Solution:** Standard Workers CPU limits apply to the sandbox. The harness should catch timeouts and retry or report the error

### "DynamicWorkerExecutor Requires LOADER Binding"

**Problem:** `DynamicWorkerExecutor` throws on construction
**Cause:** `env.LOADER` not configured in wrangler.jsonc
**Solution:** Add the LOADER binding (see [Dynamic Workers Configuration](../dynamic-workers/configuration.md))

## Configuration

Requires the same wrangler.jsonc setup as Dynamic Workers:

```jsonc
{
  "unsafe": {
    "bindings": [
      { "name": "LOADER", "type": "dynamic_worker_loader" }
    ]
  }
}
```

## See Also

- **[API](./api.md)** — API reference
- **[Dynamic Workers Gotchas](../dynamic-workers/gotchas.md)** — Sandbox-level issues
