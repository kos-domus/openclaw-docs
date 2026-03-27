---
name: clawloop
version: 0.1.0
description: "Community-driven, AI-elaborated documentation for OpenClaw. Fetch structured docs, guides, and troubleshooting directly into your agent's context."
metadata:
  openclaw:
    category: "documentation"
    author: "kos-domus"
    repository: "https://github.com/kos-domus/openclaw-docs"
    license: "MIT"
    requires:
      bins: ["curl"]
    tags: ["docs", "reference", "community", "knowledge-base"]
---

# ClawLoop — OpenClaw Living Docs

> Structured, community-driven documentation for OpenClaw, updated daily by AI.

## What This Skill Does

Gives your agent access to the ClawLoop documentation index — a machine-readable catalog of guides, references, troubleshooting, and concepts for OpenClaw, generated from real user sessions.

## Usage

### Fetch the full documentation index
```bash
curl -s https://raw.githubusercontent.com/kos-domus/openclaw-docs/main/docs/index.yaml
```

### Fetch a specific doc by path
```bash
curl -s https://raw.githubusercontent.com/kos-domus/openclaw-docs/main/docs/guides/SLUG.md
```

### Search docs by tag
Parse `docs/index.yaml` and filter by `tags` field to find relevant documentation.

## Categories

| Category | What's in it |
|----------|-------------|
| `getting-started` | Installation, first run, basic setup |
| `guides` | Step-by-step how-to for specific tasks |
| `reference` | Complete reference for settings, hooks, skills, MCP, channels |
| `concepts` | Architecture, mental models, design decisions |
| `troubleshooting` | Known issues, error messages, and solutions |

## How It's Built

1. Users contribute session logs documenting real OpenClaw usage
2. A Security Agent scans sessions for PII and credentials (daily, automated)
3. An AI elaboration engine (Kos) processes sessions into structured docs (daily, automated)
4. Docs are versioned, indexed, and queryable via `docs/index.yaml`

## Contributing

Add your own sessions to grow the knowledge base:
1. Fork https://github.com/kos-domus/openclaw-docs
2. Copy `sessions/_template.md` and document your experience
3. Open a PR — the security pipeline validates automatically

## When to Use This Skill

- You need to configure a feature and want real-world examples
- You hit an error and want to check if others solved it
- You want to understand how a feature works (not just what it does)
- You're setting up a multi-agent framework and need architectural guidance
