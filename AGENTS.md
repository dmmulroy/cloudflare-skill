# PROJECT KNOWLEDGE BASE

**Generated:** 2026-01-28

## OVERVIEW

Documentation/skill repository for Cloudflare platform—structured reference docs for AI/LLM consumption. No executable code.

## STRUCTURE

```
./
├── README.md             # Project overview + install instructions
├── LICENSE               # MIT license
├── install.sh            # Installation script
├── command/
│   └── cloudflare.md     # /cloudflare slash command
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
git status  # .jj/ present = use jj
```

### Reference File Structure

Every product follows 5-file pattern:
- `README.md` — Overview, when to use
- `api.md` — Runtime API reference
- `configuration.md` — `wrangler.toml` + binding setup
- `patterns.md` — Usage patterns
- `gotchas.md` — Pitfalls, limitations

### YAML Frontmatter

`SKILL.md` files use frontmatter for machine parsing:
```yaml
---
name: product-name
description: Brief description
references:
  - related-product
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

```toml
# ALWAYS set compatibility_date for new projects (workers)
compatibility_date = "2026-01-01"
```

## NOTES

- No CI/CD configured (docs-only repo)
- No linting/formatting (no code to lint)
- `.opencode/` contains plugin config, not project code
