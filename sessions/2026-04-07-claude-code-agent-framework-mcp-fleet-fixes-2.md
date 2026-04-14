---
title: "Claude Code 5-agent framework, MCP setup, OpenClaw 2026.4.5 fleet fixes, model overhaul, secret caching"
date: "2026-04-07"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "agent-sdk", "mcp", "mcp-servers", "configuration", "troubleshooting", "security", "performance", "cron", "scheduling", "setup"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Fix all agent failures (Kos down, Kai context overflow), update OpenClaw to 2026.4.5, overhaul model configuration to GPT 5.4 primary across all agents, implement secret caching to prevent 1Password rate limit loops, build a 5-agent Claude Code native framework for the job-desk project, and install Phase 1 MCPs.

## Context
- Kos had all cron jobs failing (FailoverError)
- Kai reporting "context overflow" on every WhatsApp message
- Anthropic changed billing for third-party apps (Claude CLI)
- OpenClaw on 2026.4.2, needed update to 2026.4.5
- User purchased $200 OpenAI plan for GPT 5.4
- User wanted a Claude Code native multi-agent framework based on detailed P.IVA operating desk specs

## Steps Taken

### 1. Diagnosed Kos failures
All Kos cron jobs failing with FailoverError. Root causes:
- Anthropic billing change: third-party apps now draw from "extra usage" not plan limits
- User had to claim $100 credit at claude.ai/settings/usage
- Stale cached sessions in gateway SQLite WAL preventing recovery
- Invalid model ID `google-gemini-cli/gemini-3-flash` in default fallback chain

**Result**: Claude CLI fixed after credit claim. Removed invalid model IDs from defaults.

### 2. Overhauled model configuration for all agents
Switched all main agents to GPT 5.4 primary (user purchased $200 OpenAI plan). Removed `claude-cli/opus-4.6` from all fallback chains to prevent init failures.

Final model configuration:

| Agent | Primary | Fallback 1 | Fallback 2 |
|-------|---------|-----------|-----------|
| Kai | openai-codex/gpt-5.4 | gemini-3.1-pro-preview | gemini-2.5-flash |
| Kos | openai-codex/gpt-5.4 | gemini-3.1-pro-preview | gemini-2.5-flash |
| Master Control | openai-codex/gpt-5.4 | gemini-3.1-pro-preview | gemini-2.5-flash |
| Mr Wolf | openai-codex/gpt-5.4 | gemini-3.1-pro-preview | gemini-2.5-flash |
| Workers (4) | openai-codex/gpt-5.4-mini | gemini-2.5-flash | — |
| Docs elaboration | gemini-3.1-pro-preview (hardcoded) | — | — |

Removed hardcoded models from ALL cron job payloads so they use agent defaults. Fixed default fallback chain to remove non-existent models (`gemini-3-flash`, `gemini-2.5-flash-lite`, `gpt-5.3-codex`).

**Result**: All agents running on GPT 5.4 with Gemini fallbacks. No more Anthropic dependency.

### 3. Diagnosed and fixed Kai "context overflow"
Gateway logs revealed the real error was NOT context overflow — it was `Unknown model: google-gemini-cli/gemini-3-flash`. The "context overflow" is a generic fallback message when ALL models fail.

Additional context bloat found and fixed:
- Kai MEMORY.md: 373 → 75 lines (removed duplicated schemas, commands, folder trees)
- Kai SOUL.md: 293 → 43 lines (moved commands to procedures/ files loaded on demand)
- Removed stale files: 656 KB → 89 KB workspace total
- Session-memory hook: archived 24 old snapshots (138 KB → 21 KB)
- GWS Drive SKILL.md: 13 KB → 1.5 KB (full API ref replaced with quick reference)
- Removed unnecessary skills: gws-gmail, gws-gmail-send, gws-docs, firecrawl

Created procedures/ directory for Kai with on-demand loading:
- calendar.md, reminders.md, 1password.md, audio.md, youtube.md

**Result**: Kai session context reduced from ~47 KB to ~17 KB.

### 4. Updated OpenClaw to 2026.4.5
```bash
npm update -g openclaw
```
Updated systemd service version and description. Fixed `start-gateway.sh` to include `PATH="$HOME/.npm-global/bin:$PATH"`.

Encountered `bluebubbles` plugin version check error — systemd service was reporting old version. Fixed by updating `OPENCLAW_SERVICE_VERSION` in systemd unit file.

**Result**: OpenClaw 2026.4.5 running. Gateway healthy.

### 5. Implemented secret caching
Problem: `op-env.sh` makes 13 sequential `op read` calls to 1Password on every gateway restart. Combined with systemd `Restart=always`, this created an infinite rate-limit loop — each restart hit the limit, failed, triggered another restart.

Solution: Created `~/.openclaw/op-env-cached.sh` with actual secret values instead of `op read` calls. Regenerated with 1-second delays between reads to avoid rate limiting.

Updated `start-gateway.sh`:
```bash
if [ -f /home/kos/.openclaw/op-env-cached.sh ]; then
  source /home/kos/.openclaw/op-env-cached.sh
else
  source /home/kos/.openclaw/op-env.sh
fi
```

Cache permissions: 600. Regeneration required when secrets change in 1Password.

**Result**: Gateway startup is instant. No more 1Password API calls on restart.

### 6. Cleaned Kos workspace
Kos workspace was 6.7 MB — a 5.6 MB raw OpenClaw log file sitting in `reports/api-monitoring/_raw/`. Removed raw logs and temp files, reduced to 668 KB.

**Result**: Kos workspace 90% smaller.

### 7. Added self-improvement rules to all agents
Added "Miglioramento Continuo" section to all 4 main agent SOUL.md files (Kai, Kos, Master Control, Mr Wolf):
- Review daily logs, identify recurring failures
- Save patterns as memories
- Keep workspace clean (delete temp > 7 days)
- MEMORY.md hard cap: 100 lines

**Result**: Agents now have continuous improvement protocol.

### 8. Created weekly workspace health monitor
New Kos cron job: every Monday at 3pm Rome. Scans all 4 agent workspaces and reports sizes, file counts, top 5 heaviest files, stale files, alerts if MEMORY.md > 100 lines or workspace > 500 KB.

**Result**: Automated workspace bloat detection.

### 9. Cleaned duplicate Kai reminders
Found 28 duplicate cron jobs (23x "universita di Verona", 5x "portare Mar a scuola", 1x "visita medica"). Removed all duplicates keeping one of each.

**Result**: Clean cron job list.

### 10. Updated Kos cron schedules
- Security monitor: daily at 4am Rome (was 6am)
- GitHub Trending: Mon+Fri at 5am Rome (was Mon/Wed/Fri at 2am UTC)

**Result**: Better load distribution, reduced API usage.

### 11. Configured Gemini CLI with OAuth subscription
Set up Gemini CLI (`~/.gemini/settings.json`) with `authMode: "oauth-personal"` for Google subscription-based access. User authenticated via browser on mini PC.

Initially tried `authMode: "api-key"` which works but uses pay-per-use credits instead of subscription.

**Result**: Gemini CLI uses subscription, not API credits.

### 12. Built 5-agent Claude Code native framework
Created a multi-agent framework at `~/job-desk/.claude/agents/` for an "AI-native operating desk" for a solo tech P.IVA. Five agents as Claude Code sub-agents:

| Agent | File | Model | Can Spawn | Domain |
|-------|------|-------|-----------|--------|
| Co-CEO | co-ceo.md | Opus | All 4 others | Strategy, prioritization, business model, advisory |
| CSO | cso.md | Opus | None | Security, risk, compliance, VETO power |
| Frontend Specialist | frontend-specialist.md | Sonnet | None | UI/UX, design systems, AI interaction, accessibility |
| Backend Expert | backend-expert.md | Sonnet | None | APIs, data, integrations, reliability, cost optimization |
| Orchestration Architect | orchestration-architect.md | Opus | None | Workflows, memory, tool routing, evaluation, governance |

Key design decisions:
- Co-CEO is the coordinator — delegates to specialists, synthesizes results
- CSO has VETO power on security/risk decisions
- Orchestration Architect owns "how things work together"
- All agents share cross-cutting competencies: business literacy, AI literacy, FinOps, security baseline
- Handoff protocols defined between all pairs
- Agents live in `.claude/agents/` directory (Claude Code native format)

**Result**: Framework ready to use via Agent tool in any job-desk session.

### 13. Installed Phase 1 MCPs
Installed MCP servers for the Claude Code framework:

| MCP | Command | Purpose |
|-----|---------|---------|
| GitHub | `@modelcontextprotocol/server-github` | PRs, issues, code search |
| Filesystem | `@modelcontextprotocol/server-filesystem` | Extended file ops across job-desk/ |

Config at `~/job-desk/.mcp.json`. GitHub token referenced via env var `${GITHUB_PAT}`, not plaintext. Added `.mcp.json` to `.gitignore`.

PostgreSQL MCP deferred — no database running yet.

**Result**: MCPs configured, available to all agents on next session restart.

### 14. Comprehensive agent fleet analysis
Produced detailed specs for all 8 OpenClaw agents:
- Full configuration: ID, model, channel, tools, skills, Drive access, cron jobs, SOUL.md personality
- Extra .md files inventory: AGENTS.md, BOOT.md, HEARTBEAT.md, IDENTITY.md per agent
- Cross-agent patterns: communication channels, model selection strategy, Drive organization, security model
- Workers (CSO, Frontend, Backend, Orch. Architect) have AGENTS.md with domain mandates but generic SOUL.md templates

**Result**: Complete fleet documentation for replication/reference.

## Configuration Changes
- `~/.openclaw/openclaw.json` — all agent models to GPT 5.4, defaults fixed, invalid models removed
- `~/.openclaw/cron/jobs.json` — hardcoded models removed, 28 duplicate reminders deleted, schedules updated
- `~/.openclaw/start-gateway.sh` — PATH added, cached secrets support
- `~/.openclaw/op-env-cached.sh` — new, 14 cached secrets for instant startup
- `~/.config/systemd/user/openclaw-gateway.service` — version updated to 2026.4.5
- `~/.openclaw/workspace-family/SOUL.md` — slimmed to 43 lines
- `~/.openclaw/workspace-family/MEMORY.md` — slimmed to 75 lines
- `~/.openclaw/workspace-family/procedures/` — new: 5 procedure files
- `~/.openclaw/workspace-family/skills/` — removed 4 unnecessary skills, trimmed gws-drive SKILL.md
- `~/.openclaw/workspace-family/memory/` — archived 24 session snapshots
- `~/.openclaw/workspace-cos/` — removed 5.6 MB raw log
- All 4 agent SOUL.md files — added Miglioramento Continuo section
- `~/.gemini/settings.json` — configured OAuth personal auth
- `~/job-desk/.claude/agents/` — new: 5 agent definition files
- `~/job-desk/.mcp.json` — new: GitHub + Filesystem MCPs
- `~/job-desk/.gitignore` — added .mcp.json
- Docs elaboration cron job — switched from claude-cli/opus-4.6 to gemini-3.1-pro-preview

## Key Discoveries
- **"Context overflow" is misleading**: OpenClaw shows this generic error when ALL models fail, regardless of actual cause. Always check gateway logs at `/tmp/openclaw/openclaw-YYYY-MM-DD.log` for the real error.
- **Invalid model IDs poison the entire fallback chain**: `gemini-3-flash` doesn't exist. Having it in defaults caused all agents to fail even when their specific model config was correct.
- **`claude-cli/opus-4.6` init failures are sticky**: A failed Claude CLI session gets cached by the gateway. Even after fixing the root cause, the gateway retries the cached failed session. Clearing SQLite WAL + restart is needed.
- **1Password service account rate limits are aggressive**: 13 sequential `op read` calls × multiple restarts = permanent rate limit loop with systemd `Restart=always`. Solution: cache secrets locally.
- **systemd service version matters**: The `OPENCLAW_SERVICE_VERSION` env var in the systemd unit is used by plugins for compatibility checks. Must be updated on every OpenClaw update.
- **Session-memory hook creates unbounded growth**: Every `/new` or `/reset` saves a snapshot to `memory/`. Must be archived/cleaned periodically.
- **Skill SKILL.md files count toward context**: The gws-drive SKILL.md was 13 KB loaded every session. Most agents only use 3-4 commands from it.
- **Anthropic billing change**: Third-party apps (like OpenClaw using Claude CLI) now use "extra usage" credits, not plan limits. Must claim credit at claude.ai/settings/usage.
- **Claude Code custom agents**: Defined as markdown files with YAML frontmatter in `.claude/agents/`. Support model override, tool restrictions, subagent spawning, MCP server access, and effort level.
- **Gemini CLI auth modes**: `api-key` uses pay-per-use API credits. `oauth-personal` uses Google subscription (Gemini Pro/Ultra). Must use OAuth for subscription billing.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| Kos FailoverError: all models failed | Anthropic billing change + invalid model IDs | Claim credit, remove invalid models, remove claude-cli from fallbacks |
| Kai "context overflow" | Invalid `gemini-3-flash` in defaults + bloated workspace | Fixed defaults, slimmed workspace from 47 KB to 17 KB context |
| Gateway won't start after npm update | systemd service had old OPENCLAW_SERVICE_VERSION | Updated version in systemd unit file |
| 1Password rate limit infinite loop | 13 op reads per restart × Restart=always | Created op-env-cached.sh with static values |
| Gateway SECRETS_RELOADER_DEGRADED | Env vars empty from failed op reads | Cached secrets file with 1-sec delays between reads during generation |
| `bluebubbles` plugin version error | systemd reporting 2026.4.2 while binary is 2026.4.5 | Updated OPENCLAW_SERVICE_VERSION in systemd unit |
| `openclaw cron run` GatewaySecretRefUnavailableError | CLI process can't read gateway token | Use op-env-cached.sh or pass token explicitly |
| Docs elaboration keeps skipping | Model falls back to Flash which does lightweight runs | Hardcoded gemini-3.1-pro-preview for docs job |
| Duplicate reminders (28 copies) | Kai creating cron jobs without dedup checking | Cleaned duplicates, keeping one of each |
| GitHub PAT in .mcp.json plaintext | `claude mcp add` writes token directly | Replaced with `${GITHUB_PAT}` env var reference |

## Final State
- **OpenClaw**: 2026.4.5, all agents on GPT 5.4 with Gemini fallbacks, no Anthropic dependency
- **Kai**: Working on WhatsApp, slim context (17 KB), procedures on demand
- **Kos**: All cron jobs healthy, GPT 5.4 primary, workspace cleaned
- **All agents**: Self-improvement rules, workspace health monitoring active
- **Secret caching**: Instant gateway startup, no 1Password rate limit risk
- **Claude Code framework**: 5-agent operating desk (Co-CEO, CSO, Frontend, Backend, Orch. Architect) built at `.claude/agents/`
- **MCPs**: GitHub + Filesystem installed, PostgreSQL deferred
- **Gemini CLI**: OAuth subscription auth configured

## Open Questions
- Anthropic extra usage credit: monitor depletion rate for docs elaboration
- Should op-env-cached.sh be auto-regenerated weekly?
- Kos docs elaboration keeps running on Flash instead of Pro — model override in cron payload may not be respected by gateway
- PostgreSQL MCP: install when first project needs database
- Claude Code Agent Teams (experimental): evaluate for inter-agent communication when needed
- Phase 2 MCPs (Sentry, Firecrawl, Slack): install when projects require them
