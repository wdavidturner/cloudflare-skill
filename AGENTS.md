# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-24

## OVERVIEW

Documentation/skill repository for Cloudflare platform — structured reference docs for AI/LLM consumption. No executable code. Compatible with the [Agent Skills](https://agentskills.io) ecosystem (`npx skills add`).

## STRUCTURE

```
./
├── README.md             # Project overview + install instructions
├── LICENSE               # MIT license
├── install.sh            # Legacy OpenCode installation script
├── command/
│   └── cloudflare.md     # /cloudflare slash command (OpenCode)
└── skills/
    └── cloudflare/
        ├── SKILL.md              # Main skill manifest + decision trees
        └── references/           # Product subdirs
            └── <product>/
                ├── README.md
                ├── api.md
                ├── configuration.md
                ├── patterns.md
                └── gotchas.md
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Find a product | `skills/cloudflare/SKILL.md` | Decision trees + full index |
| Product reference | `skills/cloudflare/references/<product>/` | 5-file structure |

## CONVENTIONS

### VCS: jujutsu (NOT git)

```bash
# CORRECT
jj status
jj log
jj new
jj commit -m "msg"

# WRONG - do not use
git status  # use jj instead
```

### Reference File Structure

Every product follows 5-file pattern:
- `README.md` — Overview, when to use
- `api.md` — Runtime API reference
- `configuration.md` — `wrangler.jsonc` + binding setup
- `patterns.md` — Usage patterns
- `gotchas.md` — Pitfalls, limitations

### YAML Frontmatter

`SKILL.md` files use frontmatter for machine parsing:
```yaml
---
name: product-name
description: Brief description
---
```

## ANTI-PATTERNS

### SQL Injection (D1)

```typescript
// NEVER - string interpolation
const result = await db.prepare(`SELECT * FROM users WHERE id = ${id}`).all();

// ALWAYS - prepared statements with bind()
const result = await db.prepare("SELECT * FROM users WHERE id = ?").bind(id).all();
```

### Secrets Management

```bash
# NEVER commit .dev.vars (contains secrets)
# NEVER hardcode secrets in code
```

### Resource Management

```typescript
// ALWAYS close browser in finally block (browser-rendering)
const browser = await puppeteer.launch();
try {
  // ...
} finally {
  await browser.close();
}
```

### Configuration

```jsonc
// ALWAYS set compatibility_date for new projects (workers)
{ "compatibility_date": "2026-03-01" }
```

## NOTES

- No CI/CD configured (docs-only repo)
- No linting/formatting (no code to lint)
- `install.sh` and `command/` are legacy OpenCode support; primary install is `npx skills add`
