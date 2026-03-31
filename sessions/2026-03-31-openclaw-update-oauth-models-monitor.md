---
title: "OpenClaw v2026.3.28 update, OAuth subscription migration (OpenAI + Google), model selection per agent, OpenClaw Release Monitor"
date: "2026-03-31"
author: "kos-domus"
status: "ready"
tags: ["configuration", "security", "api", "multi-agent", "performance", "cron", "scheduling", "setup", "troubleshooting"]
openclaw_version: "2026.3.28"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
1. Update OpenClaw from 2026.3.23-2 to latest stable (v2026.3.28) following Kos's Release Monitor report
2. Migrate all agents from per-token API billing to OAuth subscription-based authentication (OpenAI Codex + Google Gemini CLI)
3. Add OpenClaw Release & Ecosystem Monitor as a new daily cron job for Kos
4. Add task-based model selection guides to all four main agents (Kos, Kai, Master Control, Mr Wolf)

## Context
- Kos's new OpenClaw Release Monitor (created this session) flagged that we were 2 stable releases behind (2026.3.23-2 → 2026.3.28)
- 9 CVEs in 4 days (March 18-21), including CVSS 9.9 — security patches in 3.24 and 3.28
- WhatsApp echo loop fix and Telegram splitting/crash fixes directly relevant to our setup
- OpenAI Codex OAuth was already configured during the v2026.3.28 upgrade wizard
- Google Gemini CLI needed to be installed and authenticated separately with personal Google account (AI Premium subscription)

## Steps Taken

### 1. Created OpenClaw Release & Ecosystem Monitor
Created comprehensive task spec at `~/.openclaw/workspace-cos/tasks/openclaw-release-monitor.md` covering:
- GitHub releases monitoring (stable, beta, dev channels) via `gh api`
- Official blog and announcements via `web_fetch`
- Security advisories (HIGH PRIORITY — immediate alerts)
- ClawHub skills ecosystem monitoring
- Dependency and compatibility tracking (Node.js, Claude API, WhatsApp layer, soul.md architecture)
- Delta analysis: installed version vs latest available

Created cron job: daily at 06:00 Rome time, Claude CLI Opus, Telegram DM delivery.

Test run successful: Kos reported we were 2 releases behind with security fixes pending.

**Result**: First report delivered to Telegram. Identified critical update needed.

### 2. Updated OpenClaw to v2026.3.28
Following Kos's recommendation from the Release Monitor report:
```bash
systemctl --user stop openclaw-gateway
npm install -g openclaw@latest
# 2026.3.23-2 → 2026.3.28
systemctl --user start openclaw-gateway
```

Key fixes now active:
- WhatsApp echo loop fix (#54570) — the exact feedback loop Kai was experiencing
- WhatsApp group echo suppression (#53624)
- Telegram splitting at word boundaries (#56595)
- Telegram skip whitespace-only replies — prevents GrammyError 400 crash (#56620)
- Telegram replyToMessageId validation (#56587)
- Security: sandbox media dispatch bypass closed (#54034)
- Rate-limit scoped per model, not global; ladder 30s/1m/5m

Updated `docs/meta/upstream-version.yaml` and systemd service description.

**Result**: Gateway running on v2026.3.28, all channels operational.

### 3. Migrated OpenAI to Codex OAuth (subscription billing)
The v2026.3.28 upgrade wizard configured `openai-codex` with OAuth mode. Migrated all 8 agents from `openai/gpt-*` (per-token API) to `openai-codex/gpt-*` (subscription):
```
openai/gpt-5.4       → openai-codex/gpt-5.4
openai/gpt-5.4-mini  → openai-codex/gpt-5.4-mini
```

OAuth tokens managed by the gateway runtime — no plaintext token files.

**Result**: All OpenAI usage now billed through ChatGPT Plus subscription, zero per-token charges.

### 4. Installed Google Gemini CLI and authenticated with personal account
Installed Gemini CLI:
```bash
npm install -g @google/gemini-cli  # v0.35.3
```

Required setup:
1. Created GCP project (`personal-ai-oc`) under personal Google account (not bot account)
2. Enabled "Gemini API" in GCP console (APIs & Services → Library)
3. Ran `openclaw onboard --auth-choice google-gemini-cli` with `GOOGLE_CLOUD_PROJECT=personal-ai-oc`
4. Browser OAuth flow with personal Google account (AI Premium subscription)

Key distinction: **Bot account** (dedicated bot email) keeps GWS access (Drive, Calendar, Gmail). **Personal account** handles AI model billing via subscription.

Migrated all Google models:
```
google/gemini-*  → google-gemini-cli/gemini-*
```

**Result**: All Gemini usage now through Google One AI Premium subscription. Zero per-token charges.

### 5. Achieved full subscription-based billing
Final provider setup — zero per-token API costs:

| Provider | Auth Method | Subscription |
|----------|-----------|--------------|
| OpenAI | openai-codex (OAuth) | ChatGPT Plus |
| Google | google-gemini-cli (OAuth) | Google One AI Premium |
| Anthropic | claude-cli (CLI) | Max ($90/mo) |

API keys kept in 1Password (Tech vault) as fallback only:
- `OPENAI_API_KEY`, `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`

### 6. Added task-based model selection to all agents
Updated TOOLS.md for all 4 main agents (Kos, Kai, Master Control, Mr Wolf) with:
- Models organized by provider (OpenAI, Google, Anthropic)
- Task-specific recommendations (quick/standard/heavy/code)
- Provider-specific fallback chain
- API key backup reference

Each agent has tailored selection rules matching their use cases:
- **Kos**: gpt-5.4 default, opus for heavy analysis, gpt-5.3-codex for code
- **Kai**: gpt-5.4-mini default (chat), opus for audio analysis, gemini-2.5-pro for translations
- **Master Control**: gemini-3.1-pro-preview default, gpt-5.3-codex for code generation
- **Mr Wolf**: gpt-5.4 default, opus for architecture, gpt-5.3-codex for coding

### 7. Set Master Control to Gemini 3.1 Pro Preview
Changed Master Control's primary model from `openai-codex/gpt-5.4-mini` to `google-gemini-cli/gemini-3.1-pro-preview` — Google's most capable model, via subscription.

**Result**: Master Control now runs on the best available Google model at zero per-token cost.

### 8. Diagnosed Kai issues from previous session
- Kos's 8 cron errors: all historical (channel delivery error already fixed)
- WIP WhatsApp 499 errors: connection instability during gateway restarts, self-resolving
- GEMINI_API_KEY: already loaded via op-env.sh, gateway has it
- Rate limit: caused by deleted 5-min reminder cron, already resolved

Stopped supermarket deals cron job (~100k tokens/run on GPT 5.4) to save costs.

**Result**: All issues pre-resolved. No action needed.

## Configuration Changes
- OpenClaw updated: 2026.3.23-2 → **2026.3.28** (`npm install -g openclaw@latest`)
- `~/.openclaw/openclaw.json` — All `openai/` → `openai-codex/`, all `google/` → `google-gemini-cli/`
- `~/.openclaw/openclaw.json` — Master Control model: `google-gemini-cli/gemini-3.1-pro-preview`
- `~/.openclaw/openclaw.json` — Auth profiles: `openai-codex` (OAuth) + `google-gemini-cli` (OAuth)
- `~/.openclaw/workspace-cos/TOOLS.md` — Added model selection guide
- `~/.openclaw/workspace-family/TOOLS.md` — Added model selection guide
- `~/.openclaw/workspace-orchestrator/TOOLS.md` — Added model selection guide
- `~/.openclaw/workspace-wip/TOOLS.md` — Added model selection guide
- `~/.openclaw/workspace-cos/tasks/openclaw-release-monitor.md` — NEW: release monitor task spec
- `~/.openclaw/cron/jobs.json` — Added OpenClaw Release Monitor (daily 06:00), disabled supermarket deals
- `docs/meta/upstream-version.yaml` — Updated to v2026.3.28
- `~/.config/systemd/user/openclaw-gateway.service` — Version string updated
- Installed: `@google/gemini-cli` v0.35.3 (npm global)
- GCP: Created project `personal-ai-oc`, enabled Gemini API

## Key Discoveries
- **OpenAI Codex OAuth is configured during the upgrade wizard** — The `openclaw configure` command that ran with v2026.3.28 automatically set up `openai-codex` OAuth. No manual `onboard` step needed.
- **Google Gemini CLI requires a separate GCP project** — The bot's GCP project (`kos-family`) won't work because the subscription is on the personal Google account. Created `personal-ai-oc` under personal account.
- **OAuth tokens are gateway-managed** — Not stored in files you can read. The gateway handles token refresh automatically. API keys in 1Password serve as fallback only.
- **Provider migration is a global find-replace** — Changing `openai/` to `openai-codex/` across all agent configs was safe and straightforward. Same models, different auth path.
- **Three subscriptions = zero API costs** — ChatGPT Plus + Google One AI Premium + Anthropic Max covers all model usage for all 8 agents. API keys are now strictly backup.
- **Gemini CLI needs `GOOGLE_CLOUD_PROJECT` set** — Without it, onboard fails with "This account requires GOOGLE_CLOUD_PROJECT". Must create project first and enable Gemini API.
- **Agent-level model selection via TOOLS.md** — OpenClaw agents can request specific models per task. Documenting this in TOOLS.md gives agents awareness of what's available and when to use each.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| `Gemini CLI not found` during onboard | Gemini CLI not installed | `npm install -g @google/gemini-cli` |
| `This account requires GOOGLE_CLOUD_PROJECT` | No GCP project set for personal account | Created `personal-ai-oc` project, enabled Gemini API, set env var |
| openclaw.json modified externally during edits | Gateway touching config on restart | Re-read file before editing |

## Final State
- **OpenClaw**: v2026.3.28 (latest stable), all security patches applied
- **Billing**: 100% subscription-based (OpenAI Codex + Google Gemini CLI + Claude CLI). Zero per-token API costs.
- **Agents**: All 8 agents on subscription providers, with task-based model selection guides
- **Master Control**: Upgraded to Gemini 3.1 Pro Preview
- **Cron jobs**: 4 active (GitHub Trending, API Monitor, OpenClaw Monitor at 06:00; daily wrap-up at 23:30)
- **Fallback**: API keys in 1Password as backup for all providers
- **Release Monitor**: Operational, first report delivered successfully

## Open Questions
- How often do Google Gemini CLI OAuth tokens need refreshing? Is it automatic like OpenAI Codex?
- Should we add `gpt-5.3-codex` as a fallback for coding-heavy agents (CSO, Frontend, Backend)?
- Is Google One AI Premium rate-limited for Gemini 3.1 Pro? Need to monitor Master Control for 429s.
- Should we re-enable supermarket deals cron with a cheaper model (Gemini Flash via subscription)?
- The Release Monitor identified `clawsec` (prompt-security audit tool) — worth evaluating?
