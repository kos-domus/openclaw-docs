---
title: "MCP Servers and Tool Integration"
slug: "mcp-servers"
category: "reference"
tags: ["mcp", "tools", "integration", "configuration"]
sources: ["sessions/2026-03-16-google-drive-integration.md", "sessions/2026-03-22-top-skills-and-updates.md", "sessions/2026-03-25-setup-complete-drive-skills-testing.md"]
last_updated: "2026-03-29"
version: 1
---

# MCP Servers and Tool Integration

MCP (Model Context Protocol) servers provide standardized tool access for OpenClaw agents. This document covers how MCP servers work, available integrations, and configuration patterns.

## What Is an MCP Server?

An MCP server is a process that exposes tools (functions) to an AI agent through a standardized protocol. Instead of each agent implementing its own API integrations, MCP servers provide a uniform interface.

```
Agent → MCP Client → MCP Server → External Service
                                    (Google, GitHub, etc.)
```

## Google Workspace MCP

The `google-workspace-mcp` package provides direct access to Gmail, Calendar, Drive, Docs, and Sheets.

### Quick Setup

```bash
npx playbooks add skill openclaw/skills --skill google-workspace-mcp
```

### Available Tools

```bash
# Drive operations
mcporter call --server google-workspace --tool "drive.list" pageSize=20
mcporter call --server google-workspace --tool "drive.search" q="name contains 'report'"
mcporter call --server google-workspace --tool "drive.createFolder" name="Projects"
mcporter call --server google-workspace --tool "drive.upload" name="file.pdf" path="./file.pdf"
mcporter call --server google-workspace --tool "drive.share" fileId="ID" email="<example_email>" role="writer"
```

### GWS CLI vs MCP

| | GWS CLI | google-workspace-mcp |
|---|---------|---------------------|
| Setup | Manual (OAuth, Cloud Console) | Quick (playbooks) |
| Flexibility | Full API access | Curated tool set |
| Control | Granular | Simplified |
| Updates | Dynamic (API discovery) | Version-pinned |

**Recommendation**: GWS CLI for production setups that need fine-grained control. MCP for quick integration or prototyping.

## Chrome DevTools MCP

Attach to running Chrome sessions with real login credentials:

```json
{
  "mcp": {
    "servers": {
      "chrome-devtools": {
        "command": "chrome-devtools-mcp",
        "args": ["--attach"]
      }
    }
  }
}
```

This allows agents to interact with web applications as an authenticated user — useful for services that don't have APIs.

## ContextEngine Plugin Slot

OpenClaw v2026.3.13 introduced the ContextEngine plugin API with full lifecycle hooks:

| Hook | When It Fires | Use Case |
|------|--------------|----------|
| `bootstrap` | Session start | Load memory, set initial context |
| `ingest` | New message received | Pre-process, classify, route |
| `assemble` | Before generating response | Gather relevant context |
| `compact` | Context window filling up | Summarize, prune old context |
| `afterTurn` | After response sent | Save state, update memory |

This enables custom memory strategies (RAG, summarization, external vector stores) without modifying OpenClaw core.

## Security Considerations

### SSRF Protection

OpenClaw uses a **fail-closed** SSRF policy — requests to internal network addresses are blocked by default. This prevents agents from being tricked into accessing internal services.

### Webhook Hardening

Recent updates include hardened webhook validation to prevent spoofed tool callbacks.

### MCP Server Trust

MCP servers run as separate processes with their own credentials. Treat them like any other service:
- Use dedicated credentials (not your personal ones)
- Apply least-privilege access
- Monitor for unusual API call patterns
- Review MCP server code before deployment (especially third-party servers)

## Integration Patterns

### LiteLLM Proxy

For multi-agent setups, a LiteLLM proxy sits between agents and AI providers:

```
Agents → LiteLLM Proxy → Anthropic / OpenAI / Ollama
```

Benefits:
- **Agents never see real API keys** — the proxy injects them at request time
- **Rate limiting** per agent
- **Cost tracking** per agent
- **Model routing** — send different agents to different providers

### Domain Allowlist (Egress Control)

When using MCP servers that make outbound requests, control which domains are reachable:

```
Allowed: api.anthropic.com, googleapis.com, api.telegram.org
Blocked: everything else
```

This prevents data exfiltration through MCP servers that might be compromised or misconfigured.
