---
title: "Kai troubleshooting (reminders, audio, model), Kos OpenClaw Release Monitor"
date: "2026-03-30"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "troubleshooting", "cron", "scheduling", "configuration", "performance", "setup"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
Fix Kai's reminder spam, audio quality issues, model access problems, and add OpenClaw Release Monitor to Kos's daily tasks.

## Context
- Kai's reminder cron job (every 5 min) was spamming WhatsApp with "no reminders" messages
- Audio transcription quality was poor due to aggressive ffmpeg filters
- Kai couldn't access Claude CLI or write to reminders.json
- Gemini 2.5 Pro rate-limited when set as Kai's primary model
- Kos needed an OpenClaw ecosystem monitor for releases, security, and compatibility

## Steps Taken

### 1. Fixed Reminder Cron Spam
The 5-minute cron job was always delivering messages to WhatsApp, even when no reminders were due. Changed delivery mode to `none`, then updated message to say "SKIP" when empty. Still delivered because OpenClaw cron `announce` mode always sends output.

Final fix: **removed the cron job entirely**. Reminders now checked via Kai's heartbeat (HEARTBEAT.md section 4) — runs on gateway restart and scheduled heartbeat, not every 5 minutes.

**Result**: No more spam. Reminders checked organically.

### 2. Fixed Audio Quality
Original ffmpeg filter was too aggressive:
- `highpass=f=200` → cut too much bass
- `lowpass=f=3000` → cut all clarity above 3kHz (speech needs up to 8kHz)
- `afftdn=nf=-25` → heavy noise reduction distorted voice

Iterative fixes:
1. First: relaxed to `highpass=80, lowpass=8000, afftdn=-20` → better but still processed
2. Final: **removed all filters entirely** — just format conversion (16kHz mono WAV). Whisper handles noisy audio natively, filters were doing more harm than good.

Also upgraded Whisper model from `medium` to `large-v3` for better accuracy on Italian speech, noisy environments (university lectures, meetings).

**Result**: Much better transcription quality on Voice Memos recordings.

### 3. Fixed Claude CLI Access for Kai
Kai reported Claude CLI was inaccessible. Investigation found:
- `claude-cli/opus-4.6` was in Kai's **fallback** models, not primary — only used when all other models failed
- Kai tried `exec claude` but didn't know the correct syntax
- `exec` tool is NOT sandboxed for Kai (`sandbox.mode: "off"`)
- `claude` binary is at `/usr/bin/claude` and in the gateway's PATH

Fix: Added explicit `claude --print "prompt"` as an allowed exec command in TOOLS.md, so Kai knows the exact syntax for heavy analysis tasks.

**Result**: Kai can now use Claude CLI via `exec claude --print "prompt"` for audio analysis and complex tasks.

### 4. Fixed reminders.json Access
Kai reported reminders.json was not readable/writable. Investigation found the file was actually accessible via GWS (tested successfully with `gws drive files get`). The issue was Kai didn't have the exact GWS commands in his MEMORY.md.

Fix: Added exact download/upload commands with file IDs to MEMORY.md:
```bash
# Read: gws drive files get --params '{"fileId": "REMINDERS_FILE_ID", "alt": "media"}'
# Update: gws drive files update --params '{"fileId": "REMINDERS_FILE_ID"}' --upload reminders.json
```

**Result**: reminders.json confirmed working — already had 13 entries from Kai's earlier activity.

### 5. Fixed Kai Gateway Stuck
Kai stopped responding to messages. Logs showed gateway heartbeat running but `lastInboundAt` frozen and `messagesHandled` stuck at 98.

Fix: `systemctl --user restart openclaw-gateway`

**Result**: Kai responsive again after restart.

### 6. Model Chain Adjustments for Kai
Went through several iterations:
- Started: Gemini 2.5 Flash (primary)
- Tried: Gemini 2.5 Pro → hit 429 rate limit within minutes
- Final: **GPT 5.4-mini** (primary), Gemini 2.5 Pro (fallback 1), GPT 5.4 (fallback 2), Claude CLI Opus (fallback 3, heavy analysis)

**Result**: Stable model chain with cost-effective primary and quality fallbacks.

### 7. Disabled Supermarket Deals Cron
`Report offerte supermercati` was using ~100k tokens per run on GPT 5.4. Disabled to save costs.

### 8. Added OpenClaw Release & Ecosystem Monitor
Created comprehensive task file at `~/.openclaw/workspace-cos/tasks/openclaw-release-monitor.md` covering:
- GitHub releases (stable, beta, dev channels)
- Official blog and announcements
- Security advisories (high priority — immediate alert)
- ClawHub skills ecosystem
- Dependency and compatibility changes (Node.js, Claude API, WhatsApp layer, soul.md architecture)
- Third-party integrations and community patterns

Created cron job: daily at 06:00 Rome time, Claude CLI Opus, Telegram DM delivery.
Report includes: alert critici, release changelog, security status, skills updates, compatibility analysis, delta between installed and latest version, upgrade recommendation.

**Result**: Cron job registered and test-triggered.

### 9. Kos API Cost Monitoring Fix
Kos reported he couldn't access OpenAI billing data (403/404 on billing endpoints). Clarified that **OpenClaw's own cron run logs** already contain all token usage data — no need for billing API access.

Command: `openclaw cron runs --id <JOB_ID> --limit 10` returns `usage.input_tokens`, `output_tokens`, `total_tokens`, `model`, `provider`, `durationMs` per run.

This was already documented in Kos's task file but he hadn't re-read it after the update.

**Result**: Kos has the data source he needs without any API permission changes.

## Configuration Changes
- `~/.openclaw/cron/jobs.json` — Removed reminder cron, disabled supermarket deals, added OpenClaw Monitor
- `~/.openclaw/openclaw.json` — Kai model: GPT 5.4-mini primary, Gemini Pro/GPT 5.4/Claude CLI fallbacks
- `~/.openclaw/workspace-family/SOUL.md` — Added Claude CLI model preference for audio analysis
- `~/.openclaw/workspace-family/TOOLS.md` — Added `claude --print` as exec tool, Whisper venv path
- `~/.openclaw/workspace-family/MEMORY.md` — Added exact GWS commands for reminders.json read/write
- `process-audio.sh` — Removed all ffmpeg filters, upgraded Whisper to large-v3
- Gateway restarted twice (model change + stuck fix)

## Key Discoveries
- **Cron jobs with `announce` ALWAYS deliver** — Even `delivery.mode: "none"` and SKIP responses get announced. The only way to prevent delivery is to not use cron for conditional tasks. Use heartbeat instead.
- **Gemini 2.5 Pro rate limits quickly** — Free tier quota is lower than expected. Not suitable as primary model for a chatty WhatsApp agent.
- **ffmpeg filters hurt Whisper quality** — Whisper is trained on noisy real-world audio. Pre-processing filters (highpass, lowpass, noise reduction) remove information Whisper needs. Just convert format, don't filter.
- **Whisper large-v3 is essential for Italian** — Medium model misses nuances, especially with background noise. Large-v3 is significantly better for non-English speech.
- **OpenClaw cron runs contain full token usage** — No need for provider billing API access. `openclaw cron runs --id <ID>` has everything.
- **Gateway can get stuck on WhatsApp** — lastInboundAt freezes, messages stop processing. `systemctl --user restart openclaw-gateway` is the fix.
- **Kai's feedback loop** — When Kai sends a message to the WhatsApp group, the gateway can pick it up as inbound and trigger Kai to respond to himself. Need to be careful with cron delivery to groups.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| Reminder cron spamming "no reminders" | Cron `announce` always delivers output | Removed cron, use heartbeat instead |
| Gemini 429 rate limit | Too many cron runs on Gemini free tier | Switched to GPT 5.4-mini primary |
| Audio transcription poor quality | ffmpeg filters too aggressive (lowpass 3kHz) | Removed all filters, let Whisper handle raw audio |
| Kai "can't access Claude CLI" | claude-cli is a model provider, not exec command; not documented in TOOLS.md | Added `claude --print "prompt"` to TOOLS.md |
| Kai "can't write reminders.json" | Missing exact GWS commands in MEMORY.md | Added download/upload commands with file IDs |
| Gateway stuck (messages not processing) | WhatsApp connection frozen | `systemctl --user restart openclaw-gateway` |
| Kos can't access OpenAI billing | API key lacks billing permissions | Use `openclaw cron runs` instead (already has token data) |

## Final State
- **Kai**: GPT 5.4-mini primary, no more spam, audio pipeline with Whisper large-v3, reminders via heartbeat
- **Kos cron jobs**: 4 active (GitHub Trending, API Monitor, OpenClaw Monitor at 06:00; daily wrap-up at 23:30), supermarket deals disabled
- **All Kos morning reports on Claude CLI** (included in Max subscription)
- **OpenClaw Monitor**: test-triggered, awaiting delivery

## Open Questions
- Should Kai's heartbeat run on a fixed schedule (e.g., daily at 7:00 AM) instead of only on gateway restart?
- How to prevent Kai's feedback loop (responding to own messages in WhatsApp group)?
- Is GPT 5.4-mini sufficient for Kai's Italian quality, or should we consider a different primary?
- Whisper large-v3 first run will download 1.5 GB — has this completed yet?
- Should supermarket deals be re-enabled with a cheaper model (Gemini Flash)?
