# Cloudflare Skill for OpenCode

Comprehensive Cloudflare platform reference docs for AI/LLM consumption. Covers Workers, Pages, storage (KV, D1, R2), AI (Workers AI, Vectorize, Agents SDK), networking, security, and infrastructure-as-code.

## Install

Local installation (current project only):

```bash
curl -fsSL https://raw.githubusercontent.com/dmmulroy/cloudflare-skill/main/install.sh | bash
```

Global installation (available in all projects):

```bash
curl -fsSL https://raw.githubusercontent.com/dmmulroy/cloudflare-skill/main/install.sh | bash -s -- --global
```

## Usage

Once installed, the skill appears in OpenCode's `<available_skills>` list. The agent loads it automatically when working on Cloudflare tasks.

Use the `/cloudflare` command to load the skill and get contextual guidance:

```
/cloudflare set up a D1 database with migrations
```

### Updating

To update to the latest version:

```
/cloudflare --update-skill
```

## Structure

The installer adds both a skill and a command:

```
# Skill (reference docs)
skills/cloudflare/
├── SKILL.md              # Main manifest + decision trees
└── references/           # Product subdirectories
    └── <product>/
        ├── README.md         # Overview, when to use
        ├── api.md            # Runtime API reference
        ├── configuration.md  # wrangler.toml + bindings
        ├── patterns.md       # Usage patterns
        └── gotchas.md        # Pitfalls, limitations

# Command (slash command)
command/cloudflare.md     # /cloudflare entrypoint
```

### Decision Trees

The main `SKILL.md` contains decision trees for:
- Running code (Workers, Pages, Durable Objects, Workflows, Containers)
- Storage (KV, D1, R2, Queues, Vectorize)
- AI/ML (Workers AI, Vectorize, Agents SDK, AI Gateway)
- Networking (Tunnel, Spectrum, WebRTC)
- Security (WAF, DDoS, Bot Management, Turnstile)
- Media (Images, Stream, Browser Rendering)
- Infrastructure-as-code (Terraform, Pulumi)

## Products Covered

Workers, Pages, D1, Durable Objects, KV, R2, Queues, Hyperdrive, Workers AI, Vectorize, Agents SDK, AI Gateway, Tunnel, Spectrum, WAF, DDoS, Bot Management, Turnstile, Images, Stream, Browser Rendering, Terraform, Pulumi, and 40+ more.

## License

MIT - see [LICENSE](LICENSE)
