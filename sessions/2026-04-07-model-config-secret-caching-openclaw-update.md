---
title: "Model config overhaul, secret caching, OpenClaw 2026.4.5 update, context overflow diagnosis"
date: "2026-04-07"
author: "kos-domus"
status: "processed"
tags: ["configuration", "security", "troubleshooting", "performance", "multi-agent", "automation", "cron"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Fix Kos agent failures (all cron jobs down), resolve Kai context overflow, overhaul model configuration across all agents, update OpenClaw to 2026.4.5, and implement secret caching to prevent 1Password rate limit loops.

## Context
- Kos had ALL cron jobs failing with FailoverError
- Kai was reporting "context overflow" on every message
- OpenClaw was on 2026.4.2, update to 2026.4.5 available
- Anthropic changed billing: third-party apps (like OpenClaw using Claude CLI) now draw from extra usage, not plan limits

## Steps Taken

### 1. Diagnosed Kos failures
All Kos cron jobs failing with `FailoverError` at init. Root cause: Anthropic API returning "Third-party apps now draw from your extra usage, not your plan limits. Claim $100 credit at claude.ai/settings/usage."

**Fix**: User claimed credit at claude.ai/settings/usage. Claude CLI works after claiming.

### 2. Overhauled model configuration for all agents
Switched all main agents from Claude CLI primary to GPT 5.4 primary (user purchased $200 OpenAI plan).

| Agent | Primary | Fallback 1 | Fallback 2 |
|-------|---------|-----------|-----------|
| Kai | openai-codex/gpt-5.4 | google-gemini-cli/gemini-3.1-pro-preview | google-gemini-cli/gemini-2.5-flash |
| Kos | openai-codex/gpt-5.4 | google-gemini-cli/gemini-3.1-pro-preview | google-gemini-cli/gemini-2.5-flash |
| Master Control | openai-codex/gpt-5.4 | google-gemini-cli/gemini-3.1-pro-preview | google-gemini-cli/gemini-2.5-flash |
| Mr Wolf | openai-codex/gpt-5.4 | google-gemini-cli/gemini-3.1-pro-preview | google-gemini-cli/gemini-2.5-flash |
| Workers | openai-codex/gpt-5.4-mini | google-gemini-cli/gemini-2.5-flash | — |
| Docs elaboration (cron) | claude-cli/opus-4.6 (hardcoded) | — | — |

Removed `claude-cli/opus-4.6` from all agent fallback chains — it was causing init failures that poisoned the entire failover. Removed invalid model `google-gemini-cli/gemini-3-flash` from defaults (doesn't exist).

Removed hardcoded models from all cron job payloads so they use agent defaults. Only exception: docs elaboration keeps Opus hardcoded.

### 3. Diagnosed Kai "context overflow"
The "context overflow" message was misleading. Actual errors from gateway logs:
1. `Unknown model: google-gemini-cli/gemini-3-flash` (invalid model ID in default fallbacks)
2. `LLM request timed out` (workspace too large, stale sessions cached)

Real context issues found and fixed:
- Kai MEMORY.md: 373 lines (17 KB) — reduced to 75 lines (3 KB) by removing duplicated schemas, commands, and folder trees already in SOUL.md
- Kai SOUL.md: 293 lines (13 KB) — reduced to 43 lines (1.8 KB) by moving command references to `procedures/` files loaded on demand
- Stale temp files in workspace: removed transcripts, duplicate JSONs, stray images (656 KB → 89 KB total)
- Session memory hook (`session-memory`): 24 snapshots totaling 138 KB in `memory/` — archived old ones (138 KB → 21 KB)
- GWS Drive skill doc: 13 KB full API reference — replaced with 1.5 KB quick reference
- Removed unnecessary skills: gws-gmail, gws-gmail-send, gws-docs, firecrawl

### 4. Updated Kos cron schedules
- Security monitor: daily at 4am Rome (was 6am)
- GitHub Trending: Mon+Fri at 5am Rome (was Mon/Wed/Fri at 2am UTC)

### 5. Cleaned duplicate Kai reminders
Found 28 duplicate cron jobs: 23 copies of "chiamare universita di Verona", 4 copies of "portare Mar a scuola", 1 duplicate "visita medica". Removed all duplicates keeping one of each.

### 6. Updated OpenClaw to 2026.4.5
```bash
npm update -g openclaw
```
Updated systemd service version from 2026.3.31 to 2026.4.5 in `~/.config/systemd/user/openclaw-gateway.service`.
Fixed `start-gateway.sh` to include PATH for npm-global bin.

### 7. Implemented secret caching
Problem: `op-env.sh` makes 13 sequential `op read` calls to 1Password on every gateway restart. Multiple restarts triggered 1Password rate limiting, which caused a restart loop (systemd `Restart=always` kept restarting, each restart hit rate limit again).

Solution: created `~/.openclaw/op-env-cached.sh` — a static file with actual secret values instead of `op read` calls. Gateway startup is now instant (no API calls).

```bash
# start-gateway.sh now uses cached secrets
if [ -f ~/.openclaw/op-env-cached.sh ]; then
  source ~/.openclaw/op-env-cached.sh
else
  source ~/.openclaw/op-env.sh  # fallback to live 1Password reads
fi
```

Cache regeneration: `source ~/.openclaw/op-env.sh && <regenerate script>` when secrets change.
Cache file permissions: 600 (owner-only read/write).

### 8. Added self-improvement rules to all agents
Added "Miglioramento Continuo" section to all 4 main agent SOUL.md files:
- Review daily logs at end of session
- Identify recurring patterns and save as memories
- Keep workspace clean (delete temp files > 7 days)
- MEMORY.md hard cap: 100 lines

### 9. Created weekly workspace health monitor
New Kos cron job: every Monday at 3pm Rome. Scans all agent workspaces and reports:
- Total size, file count, MEMORY.md/SOUL.md line counts
- Top 5 heaviest files per agent
- Stale/temp files flagged for deletion
- Alerts if: MEMORY.md > 100 lines, workspace > 500 KB, any file > 100 KB

## Configuration Changes
- `~/.openclaw/openclaw.json` — all agent models updated, defaults fixed, invalid models removed
- `~/.openclaw/cron/jobs.json` — hardcoded models removed from payloads, 28 duplicate reminders deleted, Kos schedules updated
- `~/.openclaw/start-gateway.sh` — PATH added, cached secrets support
- `~/.openclaw/op-env-cached.sh` — new, cached secrets for instant startup
- `~/.config/systemd/user/openclaw-gateway.service` — version updated to 2026.4.5
- `~/.openclaw/workspace-family/SOUL.md` — slimmed to 43 lines, commands moved to procedures/
- `~/.openclaw/workspace-family/MEMORY.md` — slimmed to 75 lines
- `~/.openclaw/workspace-family/procedures/` — new: calendar.md, reminders.md, 1password.md, audio.md, youtube.md
- `~/.openclaw/workspace-family/skills/` — removed gws-gmail, gws-gmail-send, gws-docs, firecrawl
- `~/.openclaw/workspace-family/skills/gws-drive/SKILL.md` — replaced 13 KB full API ref with 1.5 KB quick ref
- All 4 agent SOUL.md files — added Miglioramento Continuo section

## Key Discoveries
- **"Context overflow" is a misleading generic error**: OpenClaw shows this when ALL models in the fallback chain fail, regardless of actual cause. Check gateway logs for the real error.
- **Invalid model IDs poison the entire fallback chain**: `gemini-3-flash` doesn't exist but was in defaults. When the primary model failed, it tried this invalid fallback and everything died.
- **`claude-cli/opus-4.6` init failures are sticky**: Even as a fallback, a failed Claude CLI init poisons the session. The gateway caches the failed session and keeps retrying it. Removing Claude CLI from fallbacks fixed it immediately.
- **1Password service account has aggressive rate limits**: 13 sequential `op read` calls per restart × multiple restarts = rate limit loop. Solution: cache secrets locally.
- **systemd `Restart=always` + 1Password rate limit = infinite loop**: Each restart triggers op reads, which get rate-limited, which causes startup failure, which triggers restart.
- **Session-memory hook creates unbounded growth**: Every `/new` or `/reset` saves a snapshot to `memory/`. Over time this fills the workspace and bloats context.
- **Skill docs count toward context**: The gws-drive SKILL.md was 13 KB — loaded every session. Replacing with a 1.5 KB version saved significant context.
- **Workspace files accumulate silently**: Temp files, duplicate JSONs, stale transcripts, old session snapshots — they all count toward context even if not explicitly referenced.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| Kos FailoverError: init failed | Anthropic billing change for third-party apps | Claim $100 credit at claude.ai/settings/usage |
| Kai "context overflow" | Invalid model `gemini-3-flash` in default fallbacks | Removed invalid model from defaults |
| Kai "LLM request timed out" | 225 KB workspace loaded into context | Slimmed workspace to 89 KB, SOUL.md to 43 lines |
| Gateway won't start after update | systemd service had old version (2026.3.31) | Updated OPENCLAW_SERVICE_VERSION in systemd unit |
| 1Password rate limit loop | 13 op reads per restart × Restart=always | Created op-env-cached.sh with static values |
| Gateway "SECRETS_RELOADER_DEGRADED" | Env vars empty from rate-limited op reads | Cached secrets file eliminates API calls on startup |
| `openclaw cron run` fails | CLI needs env vars but 1Password rate-limited | Use cached secrets or wait for rate limit to clear |
| Duplicate reminders (28 copies) | Kai created cron jobs without checking for existing ones | Deduplicated, keeping one of each |

## Final State
- OpenClaw 2026.4.5, all agents on GPT 5.4 primary with Gemini fallbacks
- Kai workspace optimized: 43-line SOUL.md, 75-line MEMORY.md, procedures on demand
- Secret caching eliminates 1Password dependency on restart
- Weekly workspace health monitoring active
- Self-improvement rules on all agents
- 28 duplicate reminders cleaned up

## Open Questions
- Should op-env-cached.sh be auto-regenerated periodically (e.g., weekly cron)?
- Anthropic extra usage credit ($100): how fast will it deplete with docs elaboration running on Opus?
- Should we add a context budget check to the workspace health monitor (estimate token count, not just file size)?

## ELABORATION NOTE FOR KOS
This session and the 2026-04-04 session contain rich material for creating NEW reference docs. Priority targets from the CLAUDE.md roadmap:
1. **environment-variables.md** — every env var, where it's set, which agents need it (this session has detailed 1Password/op-env content)
2. **agent-fleet-reference.md** — all agent models, channels, workspaces, scopes (this session has the full model matrix)
3. **cron-scheduling-reference.md** — complete cron syntax with env var wrapping pattern (from 2026-04-04 session)
4. **audio-pipeline-reference.md** — AssemblyAI pipeline flags and architecture (from 2026-04-04 session)
Do NOT just merge into existing docs. CREATE these as new standalone reference docs.
