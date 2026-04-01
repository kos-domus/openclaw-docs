---
title: Agent Fleet Design
slug: agent-fleet-design
category: concepts
tags:
- multi-agent
- fleet
- hierarchy
- design
sources:
- sessions/2026-03-23-cos-family-wip-agents.md
- sessions/2026-03-23-handson-8agent-setup.md
last_updated: '2026-03-30'
version: 2
---

# Agent Fleet Design

This document covers the principles and practical patterns for designing and managing a fleet of specialized OpenClaw agents across multiple channels and contexts.

## Fleet Hierarchy Pattern

A typical fleet has three layers:

```
Owner
  └─ Chief of Staff (CoS)          ← Fleet coordinator
       ├─ Orchestrator              ← Work framework dispatcher
       │   ├─ Security Agent
       │   ├─ Frontend Specialist
       │   ├─ Backend Expert
       │   └─ Architecture Agent
       ├─ Family Agent              ← WhatsApp family group
       └─ Project Agent             ← Side projects
```

### Role Definitions

| Role | Responsibility | Entry Point? |
|------|---------------|-------------|
| **Chief of Staff** | Fleet-wide coordination, agent configuration, health monitoring | Yes (DM) |
| **Orchestrator** | Dispatches work tasks to specialist agents | Yes (work group) |
| **Specialist agents** | Deep expertise in a domain (security, frontend, backend, etc.) | No (delegated to) |
| **Context agents** | Manage specific life contexts (family, projects) | Yes (dedicated group) |

### Key Principle: The Orchestrator Never Does the Work

The orchestrator's job is to **route, delegate, and synthesize** — never to execute directly. It reads the request, selects the right specialist, delegates, and combines the results.

## Per-Agent Configuration

Each agent needs a complete set of workspace files:

| File | Purpose | Per-Agent or Shared? |
|------|---------|---------------------|
| `IDENTITY.md` | Position in hierarchy, mandate | Per agent |
| `SOUL.md` | Personality and communication style | Per agent |
| `AGENTS.md` | SOPs, routing, trust levels | Per agent |
| `TOOLS.md` | Available tools and limits | Per agent |
| `USER.md` | Owner context and preferences | Shared |
| `HEARTBEAT.md` | Health check schedule | Per agent |
| `BOOT.md` | Startup sequence | Per agent |

## Tool Access by Role

Follow the principle of **least privilege**:

| Agent Type | exec | browser | web_search | read/write | sessions |
|-----------|------|---------|------------|-----------|----------|
| CoS / Admin | Yes (with `exec_ask`) | Yes | Yes | Yes | Yes |
| Work specialists | Yes (with `exec_ask`) | Yes | Yes | Yes | Yes |
| Family agent | **No** | **No** | Yes | Yes | Limited |
| Public bot | **No** | **No** | **No** | Read only | List only |

**Why no `exec` for family agents?** These agents handle sensitive personal data. Removing command execution prevents prompt injection from escalating to system compromise.

## Channel-to-Agent Mapping

A practical multi-channel setup:

| Channel | Agent | Notes |
|---------|-------|-------|
| WhatsApp group "Family" | Family agent | Only agent on WhatsApp |
| Telegram DM | CoS | Private admin access |
| Telegram group (topics) | Orchestrator → specialists | Topics for each domain |
| Telegram separate bot | Project agent | Independent bot token |

**Constraint**: WhatsApp doesn't support multi-agent on one number cleanly. Keep WhatsApp for a single context and use Telegram for everything else.

## Fleet Health Monitoring

The Chief of Staff agent can run periodic health checks:

```markdown
<!-- HEARTBEAT.md for CoS -->
## Fleet Health Check (every 30 minutes)
1. Verify all agents are responding
2. Check for configuration drift
3. Audit security posture
4. Review memory system integrity
```

### Fleet Registry

Maintain a centralized `FLEET.md` in the CoS workspace as the **single source of truth** for the fleet:

```markdown
| Agent | Status | Channel | Model | Last Health Check |
|-------|--------|---------|-------|------------------|
| family | Active | WhatsApp | sonnet | 2026-03-29 06:00 |
| orchestrator | Active | Telegram | opus | 2026-03-29 06:00 |
| cso | Active | Telegram | sonnet | 2026-03-29 06:00 |
```

## Agent Scaffolding Workflow

When adding a new agent to the fleet:

```bash
# 1. Register the agent
openclaw agents add new-agent --workspace ~/.openclaw/workspace-new-agent

# 2. Create directory structure
mkdir -p ~/.openclaw/workspace-new-agent/{memory,procedures,skills}

# 3. Write bootstrap files (AFTER openclaw agents add, not before)
# IDENTITY.md, SOUL.md, AGENTS.md, TOOLS.md, USER.md

# 4. Link required skills
ln -s ~/gws-cli/skills/gws-shared ~/.openclaw/workspace-new-agent/skills/gws-shared

# 5. Copy credentials (not shared automatically)
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/new-agent/agent/auth-profiles.json

# 6. Add binding in openclaw.json

# 7. Restart gateway
openclaw gateway restart
```

> ⚠️ **Write files AFTER `openclaw agents add`**: The command creates default template files. Writing custom files before the command risks having them overwritten.

## Practical Considerations

### Model Selection

Not every agent needs the most powerful model:

| Use Case | Recommended Model | Why |
|----------|------------------|-----|
| Complex analysis, orchestration | opus | Best reasoning |
| Routine tasks, Q&A | sonnet | Good balance of speed and quality |
| Simple routing, triage | haiku / gpt-5.4-mini | Fast and cheap |

### Cost Management

With 8 agents on the same instance, costs can escalate quickly:
- **Default to cheaper models** (sonnet or gpt-5.4-mini) for routine operations
- **Use opus selectively** — cron jobs can spawn sub-agents with opus for heavy tasks
- **Keep bootstrap files concise** — every token counts when multiplied by 8 agents

### Emergent Scope

Not every agent needs a fully defined scope on day one. An agent can start with a broad mandate and evolve:

```markdown
<!-- memory/SCOPE_EVOLUTION.md -->
Track how this agent's responsibilities develop over time.
```

## What's Next

- [Multi-Agent Architecture](multi-agent-architecture.md) — Technical implementation
- [Bootstrap Files](bootstrap-files.md) — How to write effective agent instructions
- [Workspace Isolation](workspace-isolation.md) — Security boundaries
