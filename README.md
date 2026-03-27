# OpenClaw Docs

**Community-driven, AI-elaborated documentation for [Claude Code](https://claude.ai/code) (OpenClaw).**

This is a living documentation project. Humans contribute raw work sessions. An AI agent (Kos) reads them daily, extracts knowledge, and produces structured, queryable documentation.

## How It Works

```
Contributors write session logs → sessions/
                ↓
    Kos reads & analyzes daily (cron)
                ↓
    Structured docs generated → docs/
                ↓
    Index updated → docs/index.yaml
                ↓
    Anyone can fetch & query the docs
```

## Quick Start

### Read the docs

Browse `docs/` by category:

| Category | Path | Description |
|----------|------|-------------|
| Getting Started | `docs/getting-started/` | Installation, first run, basic setup |
| Guides | `docs/guides/` | Step-by-step how-to for specific tasks |
| Reference | `docs/reference/` | Complete reference for settings, hooks, skills, MCP |
| Concepts | `docs/concepts/` | Architecture, mental models, design decisions |
| Troubleshooting | `docs/troubleshooting/` | Known issues, errors, and solutions |

### Programmatic access

Fetch the index for machine-readable doc discovery:

```bash
# Get the full index
curl -s https://raw.githubusercontent.com/kos-0/openclaw-docs/main/docs/index.yaml

# Fetch a specific doc
curl -s https://raw.githubusercontent.com/kos-0/openclaw-docs/main/docs/guides/hooks-configuration.md
```

### Use in Claude Code context

Add to your `CLAUDE.md`:

```markdown
# OpenClaw Reference
Fetch https://raw.githubusercontent.com/kos-0/openclaw-docs/main/docs/index.yaml
for the documentation index. Fetch individual docs by path as needed.
```

## Contributing

We welcome session contributions from anyone using Claude Code! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

**TL;DR:**
1. Fork the repo
2. Copy `sessions/_template.md` to `sessions/YYYY-MM-DD-your-topic.md`
3. Fill in your session details
4. Set `status: ready`
5. Open a PR

Kos will process your session in the next daily run and integrate your knowledge into the docs.

## Project Structure

```
openclaw-docs/
├── CLAUDE.md              # Instructions for Kos (elaboration engine)
├── sessions/              # Raw work session logs (input)
│   ├── _template.md       # Template for new sessions
│   └── schema.yaml        # Validation schema
├── docs/                  # Elaborated documentation (output)
│   ├── index.yaml         # Machine-readable index (fetch this first)
│   ├── getting-started/
│   ├── guides/
│   ├── reference/
│   ├── concepts/
│   └── troubleshooting/
└── changelog/
    └── CHANGELOG.md       # Log of all elaboration runs
```

## License

MIT — use freely, contribute back.
