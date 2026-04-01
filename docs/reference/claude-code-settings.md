---
title: "Claude Code Settings Reference"
slug: "claude-code-settings"
category: "reference"
tags: ["settings", "configuration", "permissions", "openclaw-json"]
sources: ["sessions/2026-03-22-top-skills-and-updates.md", "sessions/2026-03-20-workspace-concept-independent-agents.md", "sessions/2026-03-25-setup-complete-drive-skills-testing.md"]
last_updated: "2026-03-29"
version: 1
---

# Claude Code Settings Reference

This document covers the key configuration files, settings, and directory structure for OpenClaw.

## Directory Structure

```
~/.openclaw/
├── openclaw.json              ← Main configuration file
├── skills/                    ← Global skills (all agents)
├── credentials/               ← OAuth and API credentials
├── .env                       ← Environment variables
├── agents/
│   └── <agent-id>/
│       ├── agent/             ← Agent runtime directory
│       │   └── auth-profiles.json  ← Per-agent credentials
│       └── sessions/          ← Conversation history
├── workspace/                 ← Default workspace
└── workspace-<name>/          ← Custom agent workspaces
    ├── AGENTS.md
    ├── SOUL.md
    ├── USER.md
    ├── TOOLS.md
    ├── IDENTITY.md
    ├── HEARTBEAT.md
    ├── BOOT.md
    ├── memory/
    ├── procedures/
    ├── errors/
    └── skills/                ← Workspace-specific skills
```

## openclaw.json: Main Configuration

### Agents

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "model": "anthropic/claude-sonnet-4-6"
    },
    "list": [
      {
        "id": "agent-name",
        "workspace": "~/.openclaw/workspace-agent-name",
        "model": "anthropic/claude-opus-4-6",
        "tools": {
          "allowlist": ["read", "write", "web_search"],
          "denylist": ["exec", "process"]
        },
        "sandbox": { "mode": "always" }
      }
    ]
  }
}
```

### Bindings

```json
{
  "bindings": [
    {
      "type": "route",
      "agentId": "agent-name",
      "match": {
        "channel": "whatsapp",
        "accountId": "default",
        "peer": { "kind": "group", "id": "<group_jid>" }
      }
    }
  ]
}
```

### Channel Configuration

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["+1XXXXXXXXXX"],
      "groupPolicy": "allowlist",
      "autoRead": true,
      "typingIndicator": true
    },
    "telegram": {
      "accounts": {
        "bot-name": {
          "token": "BOT_TOKEN",
          "groupPolicy": "allowlist",
          "allowGroups": ["GROUP_ID"]
        }
      }
    }
  }
}
```

### Skills Configuration

```json
{
  "skills": {
    "load": {
      "watch": true
    }
  }
}
```

## Tool Policies

### Sandbox Modes

| Mode | Behavior |
|------|----------|
| `off` | No sandboxing — full system access |
| `non-main` | Sandbox for all agents except the default |
| `always` | Every session runs in an isolated sandbox |

### Tool Categories

| Tool | Description | Risk Level |
|------|-------------|-----------|
| `read` | Read files from filesystem | Low |
| `write` | Write files to filesystem | Medium |
| `exec` | Execute shell commands | High |
| `process` | Manage system processes | High |
| `browser` | Browser automation (Playwright) | Medium |
| `web_search` | Search the web | Low |
| `sessions_list` | List other agent sessions | Low |
| `sessions_history` | Read other agent histories | Medium |
| `sessions_send` | Send messages to other agents | Medium |
| `sessions_spawn` | Spawn sub-agent sessions | Medium |

### exec_ask

When `exec` is enabled, `exec_ask` adds an interactive confirmation prompt before every shell command. This prevents prompt injection from silently executing destructive commands.

**Recommendation**: Always enable `exec_ask` for any agent with `exec` access.

## CLI Commands Quick Reference

```bash
# System
openclaw status                 # System overview
openclaw dashboard              # Web dashboard
openclaw doctor                 # Health check
openclaw logs --follow          # Real-time logs

# Agents
openclaw agents list            # List agents
openclaw agents list --bindings # List with channel bindings
openclaw agents add <id> --workspace <path>  # Register agent
openclaw agents bind --agent <id> --bind <channel>  # Bind to channel

# Gateway
openclaw gateway restart        # Restart the gateway
openclaw gateway stop           # Stop the gateway

# Channels
openclaw channels login --channel whatsapp --qr-terminal
openclaw channels whatsapp groups  # List WhatsApp groups

# Cron
openclaw cron add --name "Job" --cron "0 7 * * *" --message "..."
openclaw cron list
openclaw cron remove <name>

# Security
openclaw security audit         # Flag insecure settings
openclaw sandbox explain        # Show sandbox policy per session

# Updates
openclaw update --tag main      # Update from GitHub main
```

## File Permissions

| Path | Recommended | Notes |
|------|-------------|-------|
| `~/.openclaw/` | `700` | Contains credentials |
| `~/.openclaw/credentials/` | `700` | OAuth tokens |
| `~/.openclaw/agents/*/agent/auth-profiles.json` | `600` | Per-agent credentials |
| `~/.claude/` | `700` | Claude Code config (defaults to 775 — fix manually) |
| `~/.op-service-account-token` | `600` | 1Password service account token |

> ⚠️ **Security note**: `~/.claude` defaults to `775` (world-readable) on fresh installations. Fix with `chmod 700 ~/.claude`.
