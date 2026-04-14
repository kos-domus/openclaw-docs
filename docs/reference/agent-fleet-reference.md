---
title: "Agent Fleet Reference"
slug: "agent-fleet-reference"
category: "reference"
tags: ["agents", "multi-agent", "models", "reference", "fleet"]
sources:
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
last_updated: "2026-04-14"
version: 1
---

# Agent Fleet Reference

## Main OpenClaw fleet pattern

The processed sessions converged on this pattern:

| Group | Primary model | Fallbacks |
|---|---|---|
| Main agents | `openai-codex/gpt-5.4` | Gemini subscription models |
| Worker agents | `openai-codex/gpt-5.4-mini` | `gemini-2.5-flash` |
| Docs elaboration | explicit higher-quality model override when needed | minimal fallback |

## Why this changed

Anthropic CLI failures and invalid Gemini model IDs poisoned fallback chains. The fix was to standardize on GPT-5.4 as primary and use valid Gemini fallbacks only.

## Example fleet roles

| Role | Responsibility |
|---|---|
| Kai | Family-facing assistant work |
| Kos | Operations and maintenance |
| Master Control | Coordination and routing |
| Mr Wolf | Specialized execution |
| CSO / Frontend / Backend / Orchestration workers | Domain-specific subagent work |

## Operational lessons

- Invalid fallback model IDs can break the whole fleet.
- Large workspaces and oversized skill docs inflate context for every run.
- Agents need hard boundaries on workspace size, memory size, and loaded docs.
- Continuous-improvement and workspace-cleanup rules belong in agent bootstrap files, not in ad hoc operator memory.

## Claude Code side fleet

A separate five-agent Claude Code framework was also documented:

- Co-CEO
- CSO
- Frontend Specialist
- Backend Expert
- Orchestration Architect

The important reusable pattern is not the exact org chart. It is the split between coordination, security veto, implementation specialists, and orchestration governance.
