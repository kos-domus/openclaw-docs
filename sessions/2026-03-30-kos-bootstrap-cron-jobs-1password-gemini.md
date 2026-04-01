---
title: "Kos bootstrap config, cron jobs (GitHub Trending + API Monitoring), 1Password integration, Gemini validation"
date: "2026-03-30"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "cron", "scheduling", "automation", "configuration", "claude-md", "security", "api", "performance", "troubleshooting"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
1. Configure Kos agent bootstrap files (SOUL.md, HEARTBEAT.md, MEMORY.md) to enable the documentation elaboration pipeline
2. Add two new cron jobs: GitHub Trending Intelligence and API Usage & Cost Monitoring
3. Verify 1Password integration and Gemini API availability
4. Diagnose and fix OpenAI credit consumption caused by failing cron jobs

## Context
- Kos (Chief of Staff agent, id: `cos`) had zero knowledge of the openclaw-docs project — his bootstrap files only covered fleet management
- The elaboration pipeline ran once manually (2026-03-29) but was not automated
- OpenClaw agents use SOUL.md/TOOLS.md/AGENTS.md/HEARTBEAT.md/MEMORY.md for instructions — NOT CLAUDE.md (which is a Claude Code convention)
- 1Password was fully integrated in a previous session with a service account and `op-env.sh` for secret loading
- The existing "Daily session wrap-up" cron job had 6 consecutive failures eating OpenAI credits

## Steps Taken

### 1. Verified pipeline state
Checked session statuses, changelog, and docs index. Found:
- 21 sessions, all still `status: ready` (not flipped to `processed`)
- Kos had run once (March 29) producing 21 docs, but wasn't configured to run automatically
- No references to openclaw-docs in any of Kos's workspace bootstrap files

**Result**: Confirmed Kos had no automated elaboration capability.

### 2. Updated Kos bootstrap files for documentation engine

**SOUL.md** — Added `## Documentation Engine` section after `## Domain`:
```markdown
## Documentation Engine
Oltre alla fleet, gestisci la knowledge base in `/home/kos/job-desk/openclaw-docs/`.
- Leggi le sessioni in `sessions/` con `status: ready`
- Elabori docs strutturati in `docs/` seguendo Diátaxis
- Aggiorna `docs/index.yaml` dopo ogni elaborazione
- Logga in `changelog/CHANGELOG.md`
- Flippa status: `ready` → `processed`
- Docs SEMPRE in US English
- Merge, non duplicare
```

**HEARTBEAT.md** — Added `### 5. Knowledge Base Elaboration` checklist:
- Check for `status: ready` sessions
- Elaborate docs if found
- Update index, changelog
- Flip session status
- PII verification scan

**MEMORY.md** — Added `## Progetto openclaw-docs` section with paths, repo URL, current state, and conventions.

**Result**: Kos successfully ran elaboration when prompted. All 21 sessions flipped to `processed`. Kos also performed an editorial sanitization pass normalizing placeholder identifiers across docs.

### 3. Triggered Kos elaboration test
Sent Kos a direct message via Telegram telling him to check for ready sessions and follow SOUL.md Documentation Engine instructions.

**Result**: Kos executed two passes:
1. Editorial sanitization — normalized placeholder IDs across 6 docs
2. Merge pass — 16 docs updated with additional sources, 5 mapped as coverage-only, all sessions flipped to `processed`

### 4. Created GitHub Trending Intelligence cron job
Saved full task instructions to `~/.openclaw/workspace-cos/tasks/github-trending-intelligence.md`.

Report structure:
- Top 5 Fastest Growing Repos (24h)
- 5 Hidden Gems (high relative growth)
- 1 Deep Dive (most significant repo)
- Weekly Radar (Mondays only)

Created cron job: `0 6 * * * Europe/Rome`, agent `cos`, delivery via Telegram DM.

**Result**: Test run successful. Report delivered to Telegram in ~3.5 minutes. First report featured bytedance/deer-flow at #1.

### 5. Created API Usage & Cost Monitoring cron job
Saved full task instructions to `~/.openclaw/workspace-cos/tasks/api-usage-cost-monitoring.md`.

Report includes:
- Per-agent token/cost breakdown
- Tier & quota status
- Security snapshot
- Weekly analytical report (Mondays)
- Real-time alert system (cost spikes, quota limits, auth failures)

Created local data persistence structure:
```
~/.openclaw/workspace-cos/reports/api-monitoring/
├── daily/    (.json + .md)
├── weekly/   (.json + .md)
├── monthly/  (.json + .md)
├── alerts/   (.json)
├── csv/      (.csv)
└── annual/   (.json + .md)
```

**Result**: Test run successful. Report delivered to Telegram.

### 6. Diagnosed OpenAI credit drain
Found the "Daily session wrap-up" cron job had:
- 6 runs, all failed (channel delivery error: `"channel": "last"` doesn't work with multiple channels)
- **698,117 tokens consumed** on GPT-5.4 despite all failures
- Each failed run: ~100-130k input tokens, ~3.5k output tokens

**Result**: Identified as the primary cost drain.

### 7. Fixed failing daily wrap-up job
- Changed delivery from `"channel": "last"` to `"channel": "telegram", "to": "TG_USER_ID"`
- Reset `consecutiveErrors` to 0

**Result**: Channel delivery error resolved.

### 8. Optimized model assignments across all cron jobs
Switched jobs to cost-appropriate models:

| Job | Before | After |
|-----|--------|-------|
| Daily session wrap-up | gpt-5.4 (failing) | google/gemini-2.5-flash |
| GitHub Trending | gpt-5.4 | claude-cli/opus-4.6 |
| API Monitoring | gpt-5.4 | claude-cli/opus-4.6 |
| Rifiuti reminder | gpt-5.4 | unchanged (Kai agent) |
| Report offerte | gpt-5.4 | unchanged (Kai agent) |

Used `openclaw cron edit <id> --model <model>` for each change.

**Result**: Claude CLI runs on Max subscription (no per-token cost). Gemini Flash is near-free. OpenAI usage now limited to Kai's family jobs only.

### 9. Updated API Monitoring task with cron cost data sources
Added to task instructions:
- Exact command: `openclaw cron runs --id <JOB_ID> --limit 10`
- All 5 job IDs listed in a table
- Per-provider pricing reference (GPT-5.4: ~$15/$60 per M tokens, Gemini Flash: ~$0.10/$0.40, Claude CLI: included in Max)
- Instructions to check interactive sessions too

**Result**: Next API Monitoring report will include full cron job cost breakdown.

### 10. Validated Gemini API
Tested Gemini models via direct API call:
- `gemini-3-flash`: NOT available (404 — not yet in Google API)
- `gemini-2.0-flash`: deprecated for new users
- `gemini-2.5-flash`: working

Fixed daily wrap-up from `gemini-3-flash` to `gemini-2.5-flash`.

**Result**: Gemini 2.5 Flash confirmed operational. Daily wrap-up model corrected.

## Configuration Changes
- `~/.openclaw/workspace-cos/SOUL.md` — Added Documentation Engine section
- `~/.openclaw/workspace-cos/HEARTBEAT.md` — Added Knowledge Base Elaboration checklist (section 5), renumbered Report to section 6
- `~/.openclaw/workspace-cos/MEMORY.md` — Added openclaw-docs project context
- `~/.openclaw/cron/jobs.json` — Added 2 new jobs (GitHub Trending, API Monitoring), fixed daily wrap-up channel + model
- `~/.openclaw/workspace-cos/tasks/github-trending-intelligence.md` — NEW: full task instructions
- `~/.openclaw/workspace-cos/tasks/api-usage-cost-monitoring.md` — NEW: full task instructions with cron data sources
- `~/.openclaw/workspace-cos/reports/api-monitoring/` — NEW: directory structure for data persistence
- `~/.openclaw/workspace-cos/memory/reports/github-trending/` — NEW: trending report storage
- `docs/meta/upstream-version.yaml` — Formalized upstream sources as permanent with executable check methods

## Key Discoveries
- **CLAUDE.md doesn't control OpenClaw agents** — OpenClaw uses its own bootstrap system (SOUL/TOOLS/AGENTS/HEARTBEAT/MEMORY.md). CLAUDE.md is Claude Code only. This is a critical distinction.
- **Failed cron jobs still consume tokens** — The daily wrap-up burned ~700k GPT-5.4 tokens across 6 failed runs. The work executes fully, only delivery fails. Always set explicit channel/destination.
- **`openclaw cron edit --model` is the override** — No need to change agent config; model can be overridden per-job.
- **`openclaw cron run <id>` needs env vars** — Running from a shell without 1Password env vars fails. Must `source ~/.openclaw/op-env.sh` first, or let the gateway handle it.
- **`gemini-3-flash` not yet available** — Configured in openclaw.json but returns 404. Use `gemini-2.5-flash` as the current Flash model.
- **Task instructions as files > inline payloads** — Storing full instructions in `tasks/*.md` and referencing them from cron messages is cleaner and more maintainable than cramming everything into the JSON payload.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| Daily wrap-up: "Channel is required when multiple channels are configured" | `delivery.channel` set to `"last"` but no previous session channel | Set explicit `"channel": "telegram", "to": "TG_USER_ID"` |
| `openclaw cron run` → GatewaySecretRefUnavailableError | CLI shell missing env vars that gateway has | `source ~/.openclaw/op-env.sh` before running |
| `gemini-3-flash` → 404 Not Found | Model not yet available in Google API v1beta | Use `gemini-2.5-flash` instead |
| `gemini-2.0-flash` → "no longer available to new users" | Model deprecated by Google | Use `gemini-2.5-flash` |
| Kos not elaborating docs automatically | No documentation instructions in bootstrap files | Added Documentation Engine to SOUL.md + HEARTBEAT.md checklist |
| API Monitoring report missing cron job costs | Task instructions didn't mention `openclaw cron runs` as data source | Added cron data source section with all job IDs and commands |

## Final State
- **Kos bootstrap**: Fully configured for fleet management + documentation elaboration
- **Cron jobs**: 5 active jobs, all with correct channels and cost-optimized models
- **Models**: Daily wrap-up on Gemini 2.5 Flash, GitHub Trending + API Monitoring on Claude CLI (Max subscription), Kai jobs on GPT-5.4
- **Pipeline**: Working end-to-end — sessions elaborated, docs updated, index refreshed, statuses flipped
- **Gemini API**: Validated and working (2.5-flash)
- **1Password**: Service account operational, all secrets loading via `op-env.sh`

## Open Questions
- Should Kai's family jobs (Rifiuti, Offerte) also move off GPT-5.4 to save costs?
- How to handle cron job model fallback when primary model is unavailable?
- The API Monitoring job used 267k tokens on its test run — is this sustainable daily on Claude CLI Max?
- When `gemini-3-flash` becomes available, should we auto-upgrade or stay on 2.5-flash?
- Should Kos run the elaboration pipeline on a fixed cron schedule, or only when triggered by heartbeat?
