---
title: 'Bootstrap Files: CLAUDE.md, AGENTS.md, SOUL.md, and More'
slug: bootstrap-files
category: concepts
tags:
- bootstrap
- claude-md
- agents-md
- soul-md
- configuration
sources:
- sessions/2026-03-21-soul-user-md-multiagent-framework.md
- sessions/2026-03-23-handson-8agent-setup.md
last_updated: '2026-03-30'
version: 2
---

# Bootstrap Files: CLAUDE.md, AGENTS.md, SOUL.md, and More

Bootstrap files are markdown documents placed in an agent's workspace that get **injected into the agent's prompt at the start of every session**. They are the primary mechanism for shaping agent behavior, personality, and knowledge.

## The Bootstrap File Family

| File | Purpose | Scope |
|------|---------|-------|
| `CLAUDE.md` | Project-level instructions and constraints | Project |
| `AGENTS.md` | Agent role, rules, SOPs, and routing | Per agent |
| `SOUL.md` | Personality, tone, and behavioral style | Per agent |
| `USER.md` | User context, preferences, and background | Shared across agents |
| `TOOLS.md` | Available tools and usage guidelines | Per agent |
| `IDENTITY.md` | Position in hierarchy and mandate | Per agent |
| `HEARTBEAT.md` | Periodic health check schedules | Per agent |
| `BOOT.md` | Startup sequence and first-message format | Per agent |

## AGENTS.md: Role and Rules

This is the most important file for defining what an agent **does** and how it **operates**.

A well-structured AGENTS.md includes:
- **Role definition**: What this agent is responsible for
- **Routing table**: Which types of requests go where (especially for orchestrators)
- **Standard operating procedures**: Step-by-step workflows for common tasks
- **Trust levels**: What the agent can do autonomously vs. what requires confirmation
- **Hard limits** ("Won't do" list): Actions the agent must never take

### Advisory vs Enforcement

AGENTS.md rules are **advisory** — the model is instructed to follow them but is not technically prevented from violating them. For hard security boundaries, use tool policies in `openclaw.json`:

```json
{
  "id": "family-bot",
  "tools": {
    "denylist": ["exec", "process"]
  }
}
```

## SOUL.md: Personality

SOUL.md defines **how** the agent communicates, not what it does. This separation allows you to change an agent's personality without altering its capabilities.

Effective SOUL.md files include:
- **Tone and style**: Formal, casual, technical, friendly
- **Opinions**: Strong takes on the agent's domain (makes interactions more natural)
- **Conciseness preferences**: How verbose or brief the agent should be
- **Language**: Primary language and code-switching rules

### Example Personality Archetypes

- **Orchestrator**: Diplomatic but decisive, sees the big picture, never does the work itself
- **Security agent**: Appropriately cautious, direct, "if it's not verified, it doesn't ship"
- **Frontend specialist**: Opinionated about design, aesthetic but functional
- **Backend expert**: Pragmatic, values working code, prefers proven patterns

## USER.md: Shared Context

USER.md is the **same file shared across all agents**, providing consistent user context. It includes:

- User's background and expertise level
- Communication preferences
- Key contacts and relationships
- Decision-making authority

**Why it matters**: An agent explaining code to a senior engineer should communicate differently than one explaining to a beginner. USER.md enables this calibration.

## Error Learning and Procedure Memory

Two powerful patterns for agents that improve over time:

### Error Learning

When an agent makes a mistake, it documents it:

```
workspace/errors/YYYY-MM-DD-description.md
```

Each error file contains: what happened, why, how it was resolved, and how to prevent it. Agents re-read their error files at session start.

### Procedure Memory

When an agent discovers the correct approach for a frequent operation, it saves it:

```
workspace/procedures/001-deploy-frontend.md
workspace/procedures/002-database-migration.md
```

Each procedure contains step-by-step instructions, commands, gotchas, and estimated time. Agents consult procedures before attempting similar operations.

**These patterns create a cumulative improvement loop** — without them, every new session starts from zero.

## Bootstrap File Size and Cost

Every bootstrap file is loaded into the prompt at session start. This has a direct impact on:
- **Token cost**: More text = more input tokens per interaction
- **Context window**: Bootstrap files consume space that could be used for conversation
- **Latency**: Larger prompts take longer to process

**Best practice**: Keep bootstrap files concise and focused. If a file exceeds ~200 lines, consider whether all of it is necessary for every session.

## What's Next

- [Workspace Isolation](workspace-isolation.md) — Where bootstrap files live
- [Agent Fleet Design](agent-fleet-design.md) — Organizing multiple agents
