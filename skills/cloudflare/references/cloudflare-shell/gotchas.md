# @cloudflare/shell Gotchas

## Common Errors

### "SqlStorage Not Available"

**Problem:** Workspace constructor throws when accessing `sql`
**Cause:** Durable Object not configured with SQLite storage
**Solution:** Use `new_sqlite_classes` in migrations (see [Configuration](./configuration.md))

```jsonc
// ❌ Wrong — KV-only DO
{ "tag": "v1", "new_classes": ["FileAgent"] }

// ✅ Right — SQLite DO
{ "tag": "v1", "new_sqlite_classes": ["FileAgent"] }
```

### "R2 Bucket Not Bound"

**Problem:** File operations fail with binding error
**Cause:** R2 bucket not configured in wrangler.jsonc
**Solution:** Add R2 bucket binding and pass it to Workspace constructor

### "Edit Plan Rolled Back"

**Problem:** `applyEditPlan()` returns with rollback status
**Cause:** One of the edit operations failed (e.g., file not found for replace, invalid path)
**Solution:** Check the result for which operation failed. Ensure files exist before replace operations. Use `searchFiles()` to verify targets exist

```typescript
const result = await state.applyEditPlan(plan);
if (result.rolledBack) {
  console.error("Failed operation:", result.failedOperation);
}
```

### "searchFiles Returns Empty"

**Problem:** Search returns no results despite files existing
**Cause:** Glob pattern doesn't match file paths, or content query doesn't match
**Solution:** Verify glob pattern matches your file structure. Paths in the workspace start with `/`

```typescript
// ❌ May not match — missing leading path
await state.searchFiles("*.ts", "query");

// ✅ Recursive glob
await state.searchFiles("**/*.ts", "query");

// ✅ Specific directory
await state.searchFiles("src/**/*.ts", "query");
```

### "Large File Operations Slow"

**Problem:** Reading or writing large files is slow
**Cause:** Files stored in R2 have network latency per operation
**Solution:** Batch operations where possible. Use `planEdits()` + `applyEditPlan()` for multi-file changes

## Storage Limits

Workspace storage inherits Durable Object and R2 limits:

| Limit | Value | Source |
|-------|-------|--------|
| SQLite storage per DO | 10 GB | Durable Objects |
| R2 object size | 5 GB | R2 |
| R2 total storage | Unlimited | R2 (pay per use) |
| DO memory | 128 MB | Durable Objects |

## See Also

- **[API](./api.md)** — API reference
- **[Durable Objects Gotchas](../durable-objects/gotchas.md)** — DO storage limits
- **[R2 Gotchas](../r2/gotchas.md)** — R2 storage limits
