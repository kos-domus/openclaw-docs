---
title: Multi-Agent Architecture in OpenClaw
slug: multi-agent-architecture
category: concepts
tags:
- multi-agent
- architecture
- channels
- isolation
sources:
- sessions/2026-03-20-multi-agent-architecture.md
- sessions/2026-03-23-handson-8agent-setup.md
last_updated: '2026-03-30'
version: 2
---

# Multi-Agent Architecture in OpenClaw

OpenClaw supports running multiple agents on a single instance, each with isolated state, dedicated skills, and separate channel bindings. This document explains the core architecture and design decisions.

## Core Principle: Channel Agents Are Fully Independent

The fundamental design rule: **channel agents do not import anything from the core hierarchy.** Each agent is a standalone process with its own:

- Workspace directory
- Session history
- Skills
- Configuration
- Tool permissions

```
openclaw/
├── core/                          ← Hierarchical framework
│   ├── orchestrator
│   └── agents/ (search, code, data)
│
└── channels/                      ← Completely decoupled
    ├── whatsapp/
    │   ├── config.yaml            ← Agent-specific config
    │   ├── skills/                ← Agent-specific skills
    │   └── agent                  ← Standalone agent instance
    └── telegram/
        ├── config.yaml
        ├── skills/
        └── agent
```

## State Isolation

Each agent uses a **namespaced state key** to ensure conversational context never bleeds between agents or groups:

```
wa:<group_jid>       → State for WhatsApp group "Family"
wa:<group_jid>       → State for WhatsApp group "Work"
tg:<chat_id>        → State for Telegram chat
core:session:xyz    → Core agent session (never touches wa: or tg:)
```

**Why this matters**: Without state isolation, a personal assistant agent could accidentally reference conversations from a work context, or vice versa.

## Registering Agents

```bash
# Create an agent with a dedicated workspace
openclaw agents add agent-name --workspace ~/.openclaw/workspace-agent-name

# Verify registration
openclaw agents list --bindings
```

The `openclaw agents add` command automatically creates:
- The workspace directory
- A sessions directory at `~/.openclaw/agents/<id>/sessions`
- An agent directory at `~/.openclaw/agents/<id>/agent`

> ⚠️ **Write custom files AFTER `openclaw agents add`**: The command creates default template files (HEARTBEAT.md, USER.md). If you write your custom files first, they may be overwritten.

## Configuration Structure

In `openclaw.json`:

```json
{
  "agents": {
    "list": [
      {
        "id": "family",
        "workspace": "~/.openclaw/workspace-family",
        "model": "anthropic/claude-sonnet-4-6"
      },
      {
        "id": "orchestrator",
        "workspace": "~/.openclaw/workspace-orchestrator",
        "model": "anthropic/claude-opus-4-6"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "family",
      "match": {
        "channel": "whatsapp",
        "accountId": "default",
        "peer": { "kind": "group", "id": "<group_jid>" }
      }
    },
    {
      "agentId": "orchestrator",
      "match": {
        "channel": "telegram",
        "accountId": "work-bot"
      }
    }
  ]
}
```

## Tool Policies Per Agent

Each agent can have a distinct tool allowlist/denylist:

```json
{
  "id": "family-bot",
  "tools": {
    "allowlist": ["read", "write", "web_search"],
    "denylist": ["exec", "process"]
  },
  "sandbox": { "mode": "always" }
}
```

**Key insight**: Rules in AGENTS.md are **advisory** (the model can choose to ignore them). Rules in `openclaw.json` tool policies are **enforcement** (technically enforced by the runtime).

## Per-Channel Configuration

Each agent can have its own config file with model, skills, and system prompt:

```yaml
# workspace-family/config.yaml
model: claude-sonnet-4-6
maxTurns: 10
maxTokens: 2000

systemPrompt: |
  You are a family assistant for the {groupName} group.
  Respond only when mentioned.
  Use a casual, friendly tone.

skills:
  - gws-calendar
  - gws-drive

permissions:
  webSearch: false
  codeExec: false
```

## Independent Deploy

Each agent runs as a separate process, enabling independent lifecycle management:

```json
{
  "scripts": {
    "start:core": "ts-node core/orchestrator.ts",
    "start:whatsapp": "ts-node channels/whatsapp/index.ts",
    "start:telegram": "ts-node channels/telegram/index.ts"
  }
}
```

**Benefit**: You can update or restart the WhatsApp agent without affecting Telegram or the core framework.

## Practical Constraints

- **Two agents sharing a directory = broken auth and session history.** Always use `openclaw agents add` for proper scaffolding.
- **Credentials are not shared automatically** between agents. You must copy `auth-profiles.json` to each agent's directory.
- **`tools.elevated` is global** — it's based on the sender, not configurable per agent. To restrict `exec`, deny it explicitly in the agent's tool policy.
- **WhatsApp doesn't support multi-agent on one number** cleanly. Use different channels for different agents.

## What's Next

- [Workspace Isolation](workspace-isolation.md) — How workspaces provide agent boundaries
- [Bootstrap Files](bootstrap-files.md) — CLAUDE.md, AGENTS.md, SOUL.md, and friends
- [Agent Fleet Design](agent-fleet-design.md) — Designing a fleet of specialized agents
