# Cloudflare Skill

Comprehensive Cloudflare platform reference docs for AI/LLM consumption. Covers Workers, Pages, Dynamic Workers, storage (KV, D1, R2), AI (Workers AI, Vectorize, Agents SDK, Codemode), networking, security, and infrastructure-as-code.

## Install

Works with Claude Code, Cursor, Codex, OpenCode, Gemini CLI, and [39 more agents](https://github.com/vercel-labs/skills#supported-agents):

```bash
npx skills add wdavidturner/cloudflare-skill
```

### Options

```bash
# Install globally (available in all projects)
npx skills add wdavidturner/cloudflare-skill -g

# Install to specific agents
npx skills add wdavidturner/cloudflare-skill -a claude-code -a cursor

# List available skills before installing
npx skills add wdavidturner/cloudflare-skill --list
```

### Alternative: curl (OpenCode only)

```bash
# Local (current project)
curl -fsSL https://raw.githubusercontent.com/wdavidturner/cloudflare-skill/main/install.sh | bash

# Global
curl -fsSL https://raw.githubusercontent.com/wdavidturner/cloudflare-skill/main/install.sh | bash -s -- --global
```

## Usage

Once installed, the skill is available to your agent when working on Cloudflare tasks. The agent loads it automatically based on context.

For OpenCode, use the `/cloudflare` command:

```
/cloudflare set up a D1 database with migrations
```

### Updating

```bash
npx skills update
```

## Structure

```
skills/cloudflare/
├── SKILL.md              # Main manifest + decision trees
└── references/           # Product subdirectories
    └── <product>/
        ├── README.md         # Overview, when to use
        ├── api.md            # Runtime API reference
        ├── configuration.md  # wrangler.jsonc + bindings
        ├── patterns.md       # Usage patterns
        └── gotchas.md        # Pitfalls, limitations
```

### Decision Trees

The main `SKILL.md` contains decision trees for:
- Running code (Workers, Pages, Dynamic Workers, Durable Objects, Workflows, Containers)
- Storage (KV, D1, R2, Queues, Vectorize)
- AI/ML (Workers AI, Vectorize, Agents SDK, Dynamic Workers, Codemode, AI Gateway)
- Networking (Tunnel, Spectrum, WebRTC)
- Security (WAF, DDoS, Bot Management, Turnstile)
- Media (Images, Stream, Browser Rendering)
- Infrastructure-as-code (Terraform, Pulumi)

## Products Covered

Workers, Pages, Dynamic Workers, D1, Durable Objects, KV, R2, Queues, Hyperdrive, Workers AI, Vectorize, Agents SDK, AI Gateway, Codemode, Worker Bundler, Cloudflare Shell, Tunnel, Spectrum, WAF, DDoS, Bot Management, Turnstile, Images, Stream, Browser Rendering, Terraform, Pulumi, and 40+ more.

## License

MIT - see [LICENSE](LICENSE)
