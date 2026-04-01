---
title: Workspace Isolation
slug: workspace-isolation
category: concepts
tags:
- workspace
- isolation
- security
- configuration
sources:
- sessions/2026-03-20-workspace-concept-independent-agents.md
- sessions/2026-03-23-cos-family-wip-agents.md
- sessions/2026-03-23-handson-8agent-setup.md
last_updated: '2026-03-30'
version: 2
---

# Workspace Isolation

The workspace is an agent's "home" — the directory it operates from, reads instructions from, and stores persistent state in. Understanding workspace isolation is essential for building secure multi-agent systems.

## What Is a Workspace?

A workspace is a directory on the filesystem that serves as an agent's working environment. It contains:

- **Bootstrap files** (AGENTS.md, SOUL.md, TOOLS.md, USER.md) — injected into the agent's prompt at session start
- **Skills** (`skills/` subdirectory) — agent-specific capabilities
- **Memory** (`memory/` subdirectory) — persistent state between sessions
- **Procedures** (`procedures/` subdirectory) — learned workflows

**Default location**: `~/.openclaw/workspace`

**Custom location**: Set via `agents.defaults.workspace` or per-agent in `openclaw.json`.

## Workspace Is Not a Sandbox

This is the most important thing to understand:

> The workspace controls which files are **injected** into the agent's context, but it does **not** restrict filesystem access. Relative paths resolve from the workspace, but **absolute paths can reach anywhere** on the system.

For true isolation, you need OS-level controls:
- Docker containers
- Sandbox mode (`"sandbox": { "mode": "always" }`)
- Tool denylist (remove `exec`, `process`)

The workspace provides **behavioral isolation** (the agent is told to stay in its lane). Docker/sandbox provides **technical isolation** (the agent cannot escape its lane).

## Creating Isolated Workspaces

```bash
# Create agents with dedicated workspaces
openclaw agents add family --workspace ~/.openclaw/workspace-family
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add research --workspace ~/.openclaw/workspace-research
```

Each workspace gets its own:
- `memory/` — agent remembers different things
- `skills/` — agent has different capabilities
- `procedures/` — agent has learned different workflows
- Bootstrap files — agent has a different personality and role

## Skill Scope: Global vs Workspace

| Location | Scope |
|----------|-------|
| `~/.openclaw/skills/` | **Global** — available to every agent on the machine |
| `~/.openclaw/workspace-<agent>/skills/` | **Workspace** — only that specific agent |

**Best practice**: Use workspace-level skills to enforce least privilege. A family assistant doesn't need code execution skills; a code agent doesn't need email access.

```bash
# Family agent: only calendar and drive
ln -s ~/gws-cli/skills/gws-calendar ~/.openclaw/workspace-family/skills/gws-calendar
ln -s ~/gws-cli/skills/gws-drive ~/.openclaw/workspace-family/skills/gws-drive

# Work agent: all GWS skills
ln -s ~/gws-cli/skills/gws-* ~/.openclaw/workspace-work/skills/
```

## Google Drive Isolation

Google OAuth doesn't support per-folder scope — Drive access is all-or-nothing. There are three approaches to agent-level Drive isolation:

| Approach | Real Isolation? | Complexity |
|----------|----------------|-----------|
| **AGENTS.md rules** | Behavioral only | Low |
| **Selective skills** | Behavioral + modular | Low |
| **Service accounts** | Technical | Medium |

### Recommended: Selective Skills + AGENTS.md Rules

Combine skill-level control with explicit scope rules:

```markdown
<!-- In workspace-family/AGENTS.md -->
## Google Drive
You MUST operate exclusively within My Drive/Family/.
Never read, write, or list files outside this folder.
```

### Sharing Credentials

Credentials are not automatically shared between agents. Copy them manually:

```bash
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/family/agent/auth-profiles.json
```

## Key Takeaways

1. **Workspace = behavioral boundary**, not security boundary
2. **For security**, layer workspace + Docker + tool policies
3. **One workspace per agent** — never share workspaces between agents
4. **Use `openclaw agents add`** — manual directory creation risks accidental sharing
5. **Keep workspaces minimal** — every bootstrap file costs tokens at session start
6. **Git-back your workspaces** — they contain valuable learned state

## What's Next

- [Bootstrap Files](bootstrap-files.md) — Files that shape agent behavior
- [Multi-Agent Architecture](multi-agent-architecture.md) — The bigger picture
