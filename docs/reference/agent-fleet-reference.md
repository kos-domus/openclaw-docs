---
title: "Agent Fleet Reference"
slug: "agent-fleet-reference"
category: "reference"
tags: ["agents", "multi-agent", "models", "reference", "fleet", "fallback", "providers"]
sources:
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
  - "sessions/2026-04-20-openclaw-upgrade-4.15-gateway-restart.md"
  - "memory/reports/openclaw-monitor/2026-04-26.md"
  - "sessions/2026-04-26-openclaw-4.24-upgrade-bonjour-and-fleet-fallback.md"
last_updated: "2026-04-26"
version: 3
---

# Agent Fleet Reference

## Main OpenClaw fleet pattern

The processed sessions converged on this pattern:

| Group | Primary model | Fallbacks |
|---|---|---|
| Main agents | `openai-codex/gpt-5.4` | Cross-provider chain (see below), NOT Gemini-only |
| Worker agents | `openai-codex/gpt-5.4-mini` | `gemini-2.5-flash` → `anthropic/claude-sonnet-4-6` |
| Docs elaboration | explicit higher-quality model override when needed | minimal cross-provider fallback |

## Why this changed

Anthropic CLI failures and invalid Gemini model IDs poisoned fallback chains. The fix was to standardize on GPT-5.4 as primary and use valid Gemini fallbacks only — but that fix proved insufficient on 2026-04-25 when Gemini saturated and the Gemini-only fallback chain returned `All models failed` for several jobs.

## Cross-provider fallback rule (Apr 25 2026)

**A fallback chain that stays inside one provider is not a fallback.** When Gemini was saturated on 2026-04-25, jobs that had only Gemini fallbacks ran out of options and exited with `All models failed`. Same risk applies to Codex during OpenAI auth incidents.

Rule: every job important enough to warrant a fallback gets at least two distinct providers in its chain. Suggested templates:

| Tier | Primary | Fallback 1 | Fallback 2 |
|---|---|---|---|
| Critical (orchestration, oncall) | `openai-codex/gpt-5.4` | `gemini-2.5-flash` | `anthropic/claude-sonnet-4-6` |
| Standard worker | `openai-codex/gpt-5.4-mini` | `gemini-2.5-flash` | (skip) |
| Background batch | `gemini-2.5-flash` | `openai-codex/gpt-5.4-mini` | (skip) |

> ⚠️ **Don't stack two Gemini variants in the same chain.** If `gemini-2.5-pro` is the primary, a `gemini-2.5-flash` fallback gives no protection against provider-side saturation: same datacenter, same outage.

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
- Pick each agent's primary model based on the paying account or quota owner, not one global "best model" default. That keeps expensive plans attached only to the workflows that truly need them.
- Auth declarations and actual credentials are separate in multi-agent fleets. A profile stub in `openclaw.json` does not magically propagate fresh OAuth tokens into every agent workspace.
- Shared OAuth refresh tokens across multiple busy agents are a footgun. Per-agent logins or provider paths without single-use refresh-token races are safer.
- **A single-provider fallback chain is no fallback.** Critical jobs need at least two distinct providers in the chain (Apr 25 Gemini saturation incident).

## OpenAI Codex auth fragility

Codex (`openai-codex/*` models) uses a refresh-token flow that periodically fails with `refresh_token_reused` or `credential unavailable` when:

- The same refresh token is held by more than one worker process at a time.
- A long-running gateway re-uses a token that another short-lived process already rotated.
- The local cache (`~/.config/openclaw/codex-token.json` or equivalent) is stale.

Mitigations:

1. Per-agent Codex login — never share one OAuth profile across agents that fire concurrently.
2. Pair every Codex-primary job with a non-Codex fallback (see cross-provider rule above) so a single auth blip doesn't take the whole job down.
3. Schedule a daily token health check; if `refresh_token_reused` reappears, force `openclaw auth refresh codex` before the next batch run.

See [Common Errors and Solutions](../troubleshooting/common-errors.md#openai-codex-auth-failures) for the exact recovery procedure.

## Claude Code side fleet

A separate five-agent Claude Code framework was also documented:

- Co-CEO
- CSO
- Frontend Specialist
- Backend Expert
- Orchestration Architect

The important reusable pattern is not the exact org chart. It is the split between coordination, security veto, implementation specialists, and orchestration governance.
