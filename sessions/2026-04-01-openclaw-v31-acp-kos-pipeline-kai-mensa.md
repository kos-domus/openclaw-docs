---
title: "OpenClaw v2026.3.31 update, ACP status check, Kos docs pipeline automation, Kai school menu system"
date: "2026-04-01"
author: "kos-domus"
status: "processed"
tags: ["configuration", "security", "cron", "scheduling", "automation", "multi-agent", "troubleshooting", "permissions"]
openclaw_version: "2026.3.31"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
1. Update OpenClaw from v2026.3.28 to v2026.3.31 (critical security patches)
2. Check ACP (Agent Client Protocol) VS Code extension compatibility
3. Automate Kos's daily docs elaboration with a cron job and fix exec approval issues
4. Add Mar's school meal schedule to Kai with weekly WhatsApp reminders
5. Adjust cron job frequencies (API monitoring → monthly, GitHub trending → 3x/week)

## Context
- Kos's OpenClaw Release Monitor flagged v2026.3.31 with 2 critical sandbox escape CVEs
- ACP extension had bugs in v2026.3.26, user wanted to check if fixed
- Kos had 7 unprocessed sessions (status: processed) waiting for elaboration
- Kos couldn't execute shell commands via Telegram cron (exec approval required)
- User received Mar's school lunch menu PDF and wanted it integrated into Kai

## Steps Taken

### 1. Updated OpenClaw v2026.3.28 → v2026.3.31
Following Kos's Release Monitor report identifying 2 critical CVEs (CVSS 9.0):
```bash
systemctl --user stop openclaw-gateway
npm install -g openclaw@latest
# v2026.3.28 → v2026.3.31
systemctl --user start openclaw-gateway
```

Key fixes: sandbox escape via heartbeat context inheritance, sandbox escape via TOCTOU race in FS bridge, cron announce full payload preservation, cron model selection on restart, WhatsApp emoji reactions, WhatsApp outbound regressions, background tasks control plane.

**Result**: Updated and running. All channels operational.

### 2. Tested ACP extension for VS Code
Ran `openclaw acp client` both remotely (via SSH) and locally on the Mini PC. Both failed with:
```
Failed to parse JSON message: │ SyntaxError: Unexpected token '│'
```
The ACP bridge mixes its startup banner (box-drawing characters) into the JSON stream. This is the same bug from v2026.3.26 — **not fixed in v2026.3.28 or v2026.3.31**.

**Result**: ACP still broken. Parked — Kos's Release Monitor will flag when it's fixed.

### 3. Created daily docs elaboration cron job for Kos
Created `~/.openclaw/workspace-cos/tasks/daily-docs-elaboration.md` with full instructions:
- Process `status: processed` sessions into Diátaxis docs
- Verify upstream consistency (installed version vs tracked version)
- Track ALL OpenClaw version updates in changelog (including intermediate versions)
- Self-assessment after each run
- Git add, commit, push

Cron: daily at 07:00 Rome time, Claude CLI Opus, Telegram DM delivery.

**Result**: Cron job created and registered.

### 4. Fixed Kos exec approval for cron jobs
Kos reported he couldn't run `exec` commands (find, git, openclaw --version) from Telegram cron — no approval channel available.

First attempt: added `autoApprove` key to tools config → rejected by OpenClaw config validator ("Unrecognized key").

Correct fix: used `openclaw approvals allowlist` command:
```bash
openclaw approvals allowlist add --agent cos "*"
```
This gives Kos wildcard exec approval — all commands auto-approved. Stored in `~/.openclaw/exec-approvals.json`.

Also applied same fix for Kai:
```bash
openclaw approvals allowlist add --agent family "*"
```

**Result**: Both Kos and Kai can now exec any command from cron/Telegram without interactive approval.

### 5. Added OpenClaw version history to Kos's MEMORY.md
Added a version tracking table with every release installed:
- v2026.3.23-2: initial setup
- v2026.3.24: WhatsApp echo suppression, channel isolation
- v2026.3.28: 9 CVE fixes, WhatsApp echo loop, Telegram splitting
- v2026.3.31: 2 critical sandbox escapes, cron fixes, WhatsApp reactions

Rule: every upgrade adds a row to this table.

**Result**: Complete version history in Kos's memory.

### 6. Adjusted cron job frequencies
- **API Usage & Cost Monitoring**: daily → **monthly (1st of each month at 06:00)** — most usage is subscription-based now
- **GitHub Trending Intelligence**: daily at 06:00 → **Mon/Wed/Fri at 02:00** — 3x/week is sufficient

**Result**: Reduced unnecessary token usage.

### 7. Created Mar's school meal schedule system
Extracted full 8-week menu from official PDF (Euro Ristorazione, Comune locale, Scuola Primaria) into `~/.openclaw/workspace-family/memory/mensa-mar-2026.json`.

Data includes:
- 8 weeks of menus (primavera/estate A.S. 2025/2026)
- Every day: primo, secondo, contorno, frutta, pane
- Egg allergy flagged with ⚠️ (Mar is allergic to eggs)
- Weeks mapped to date ranges

Created cron job: every Monday at 07:00 → WhatsApp message with the week's menu, egg warnings highlighted, holiday awareness.

Added reference to Kai's MEMORY.md with allergen warning and cron info.

**Result**: Test run delivered Week 2 menu correctly to WhatsApp.

### 8. Investigated ElevenLabs TTS for Kai voice messages
Researched voice synthesis for Kai to send audio replies. User wanted Charlize Theron / Claudia Catani (Atomic Blonde) voice.

Findings:
- Celebrity voice cloning is blocked by ElevenLabs (No-Go Voices, consent passphrase required, forensic watermarking)
- Legally prohibited (EU personality rights, GDPR)
- Alternatives: pre-made voices (Domi, Lily), Voice Design (describe desired voice), or local sherpa-onnx-tts
- Pricing: free tier (~10 min/month), Starter $5/month (~30 min)

**Result**: Parked — user decided it's not worth it for now.

## Configuration Changes
- OpenClaw updated: v2026.3.28 → **v2026.3.31**
- `~/.openclaw/exec-approvals.json` — Wildcard `*` approval for both `cos` and `family` agents
- `~/.openclaw/cron/jobs.json` — Added "Daily docs elaboration" (Kos, 07:00 daily), added "Menu mensa Mar" (Kai, Mon 07:00), changed API monitoring to monthly, changed GitHub trending to Mon/Wed/Fri 02:00
- `~/.openclaw/workspace-cos/tasks/daily-docs-elaboration.md` — NEW: full docs elaboration task spec
- `~/.openclaw/workspace-cos/MEMORY.md` — Added OpenClaw version history table
- `~/.openclaw/workspace-family/memory/mensa-mar-2026.json` — NEW: 8-week school meal schedule
- `~/.openclaw/workspace-family/MEMORY.md` — Added mensa reference with allergen warning
- `~/.config/systemd/user/openclaw-gateway.service` — Version string updated to v2026.3.31

## Key Discoveries
- **ACP bridge still broken in v2026.3.31** — The JSON stream pollution from startup banners persists. Not a remote/SSH issue — fails locally too. Must wait for upstream fix.
- **`autoApprove` is not a valid config key** — OpenClaw uses `openclaw approvals allowlist` command, not a JSON config option. Approvals stored in `~/.openclaw/exec-approvals.json`.
- **Wildcard `*` approval needed for cron autonomy** — Without it, any exec command in a cron job times out waiting for interactive approval that never comes (no approval channel on Telegram).
- **Gateway restart required after changing approvals** — The gateway caches approval rules; adding allowlist entries without restart may not take effect.
- **v2026.3.31 install fail-closed** — Plugin/skill installs now fail if security scan finds critical issues. This caused Kai's PDF tool to need explicit approval (`openclaw approvals allowlist add --agent family "*"`).
- **School menu as structured JSON works well** — 8-week cycle mapped to date ranges, allergens flagged per dish, Kai can calculate which week to show based on current date.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| ACP `Failed to parse JSON` | Startup banner leaks into JSON stream | Bug in OpenClaw — not fixed, parked |
| Kos "exec denied approval timeout" | Cron jobs can't get interactive exec approval on Telegram | `openclaw approvals allowlist add --agent cos "*"` |
| Config invalid: "Unrecognized key: autoApprove" | `autoApprove` not a valid tools config key | Use `openclaw approvals allowlist` command instead |
| Kai "exec denied approval timeout" for PDF | v2026.3.31 fail-closed on tool usage | `openclaw approvals allowlist add --agent family "*"` + gateway restart |
| Gateway not picking up new approvals | Approval rules cached at startup | `systemctl --user restart openclaw-gateway` after allowlist changes |

## Final State
- **OpenClaw**: v2026.3.31 (latest stable, all critical patches applied)
- **ACP**: Still broken, parked for future release
- **Kos pipeline**: Automated daily at 07:00 with full exec approval. 7 sessions processed.
- **Kos cron schedule**: GitHub Trending (Mon/Wed/Fri 02:00), OpenClaw Monitor (daily 06:00), API Monitoring (1st of month 06:00), Docs Elaboration (daily 07:00), Wrap-up (daily 23:30)
- **Kai mensa**: 8-week menu in JSON, weekly Monday briefing at 07:00, egg allergy flagged
- **Exec approvals**: Wildcard `*` for both cos and family agents

## Open Questions
- When will the ACP VS Code extension bug be fixed? Kos's Release Monitor will flag it.
- Should other agents (orchestrator, wip) also get wildcard exec approval?
- The school menu is 8-week cycle — does it actually repeat after week 8 or end with the school year?
- How to handle summer when school is closed? Disable the mensa cron or let Kai report "niente mensa"?
- ElevenLabs TTS: revisit later if voice messages become a priority feature for Kai
