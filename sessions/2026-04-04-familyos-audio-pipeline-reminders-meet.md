---
title: "FamilyOS decisions/projects modules, AssemblyAI audio pipeline, reminder system fix, Meet recording workflow"
date: "2026-04-04"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "automation", "cron", "setup", "memory", "troubleshooting", "security", "scheduling"]
openclaw_version: "2026.4.2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Major infrastructure session: add decision/project tracking to FamilyOS, build a robust audio transcription pipeline with AssemblyAI, fix the broken reminder system, add YouTube transcript extraction, set up Google Meet recording workflow, and update OpenClaw with security patches.

## Context
- OpenClaw fleet running on dedicated mini PC (Ubuntu 24.04), accessed remotely via Tailscale
- Kai (family agent) on WhatsApp, Master Control on Telegram, Mr Wolf on Telegram
- FamilyOS deployed on Google Drive with 11 modules, schemas, entity graph
- Previous session left 1Password CLI integration incomplete (snap vs apt conflict resolved)
- Kai unable to handle long audio files (30+ min) — Whisper large-v3 too heavy for mini PC
- Reminder system broken after recent updates

## Steps Taken

### 1. Migrated CLAUDE.md to SOUL.md (OpenClaw native format)
OpenClaw only reads its own native `.md` files (SOUL.md, USER.md, IDENTITY.md), not CLAUDE.md. Replaced `~/job-desk/family-os/CLAUDE.md` with `SOUL.md` — pure markdown, no YAML frontmatter.

**Result**: Kai's configuration now in OpenClaw-native format. Old CLAUDE.md removed.

### 2. Added FamilyOS decision and project modules
Created two new modules to handle family research and major initiatives:

**11_decisions/** — Options evaluation pipeline:
- Folder structure: `active/`, `archive/`, `attachments/`
- Schema: `decision.schema.json` — tracks options with costs, dates, pros/cons, scores, criteria weights, source documents, status pipeline (researching → comparing → decided → archived)
- Seeded with real decision: `dec_20260402_summer_camp_mar.json`

**12_projects/** — Major initiative tracker:
- Folder structure: `active/`, `archive/`, `attachments/`
- Schema: `project.schema.json` — budget with line-item breakdown, financing sources, tax incentives (detrazioni, bonus by government level), bureaucracy checklist, milestones

Updated `entity_graph.json` with both new entities + 8 new query patterns (active decisions, deadline alerts, budget vs actual, overdue milestones, pending bureaucracy, available incentives).

**Result**: Full decision and project management integrated into FamilyOS data layer.

### 3. Added PDF intake pipeline with OCRmyPDF
Installed `ocrmypdf` + `tesseract-ocr-ita` for processing scanned PDFs:
```bash
sudo apt install ocrmypdf tesseract-ocr-ita
```
Added PDF intake workflow to Kai's SOUL.md: receive → OCR if scanned → classify → extract structured JSON → store original → link to entity graph.

OCR command for scanned documents:
```bash
ocrmypdf --language ita --rotate-pages --deskew --clean input.pdf output.pdf
```

**Result**: All agents can now process scanned PDFs with Italian OCR.

### 4. Installed YouTube transcript extraction
Installed `youtube-transcript-api` via pipx for extracting subtitles from YouTube videos without downloading audio:
```bash
pipx install youtube-transcript-api
```
Added YouTube & Podcast Intelligence section to all 6 SOUL.md files (Kai, Master Control, Mr Wolf, Kos, base, family-os). Primary method fetches subtitles instantly; fallback uses yt-dlp + Whisper for videos without captions.

**Result**: All agents can extract and summarize YouTube/podcast content.

### 5. Built AssemblyAI audio pipeline
Created a robust multi-file pipeline at `99_kai_system/scripts/audio-pipeline/`:

| File | Purpose |
|------|---------|
| `run.sh` | Entry point (activates venv) |
| `pipeline.py` | Main orchestrator (4 phases) |
| `config.py` | Settings + 1Password key reader + per-agent output dirs |
| `preprocess.py` | FFmpeg normalize + chunk (15-min segments, 10-sec overlap) |
| `transcribe.py` | AssemblyAI REST API + Whisper fallback |
| `merge.py` | Overlap deduplication |
| `summarize.py` | 4-level hierarchical summaries via Claude CLI |
| `db.py` | SQLite job/segment tracking |

Key decisions:
- Used AssemblyAI REST API directly (SDK was outdated for `speech_models` plural requirement)
- Whisper fallback only for files < 2 min (large-v3 too heavy for mini PC on longer files)
- API key cached in memory after first read to prevent 1Password session expiry mid-pipeline
- Per-agent output routing via `--agent` flag (kai/master/wolf)

Created Python venv:
```bash
python3 -m venv ~/.venvs/audio-pipeline
~/.venvs/audio-pipeline/bin/pip install assemblyai
```

Added `ASSEMBLYAI_API_KEY` to `~/.openclaw/op-env.sh` for auto-loading at session start.

**Result**: 30-min file that previously choked Kai now processes in ~4.5 minutes. Pipeline tested successfully.

### 6. Updated OpenClaw from 2026.4.1 to 2026.4.2
```bash
npm update -g openclaw
```
Security update with 2 high-severity advisories patched. Config backed up at `openclaw.json.bak-2026.4.1`.

Breaking changes checked: `x_search` and `Firecrawl web_fetch` config migration to plugin format. Current config not affected (neither feature actively configured).

**Result**: Updated with zero config changes needed.

### 7. Fixed reminder system
**Root cause analysis**: Kai was writing reminders to `reminders.json` but nothing was reading/delivering them. The working mechanism before was `openclaw cron add --at` (native one-shot cron jobs that fire at exact time and auto-delete).

**Additional issue**: `openclaw cron add` requires env vars (gateway token, etc.) but Kai's `exec` runs in a clean shell without them.

**Fix**: Updated SOUL.md with correct syntax wrapping all cron commands:
```bash
exec bash -c 'export PATH="$HOME/.npm-global/bin:$PATH" && source ~/.openclaw/op-env.sh && openclaw cron add --at "DATETIME" --tz "Europe/Rome" ...'
```

Restored full reminder management in SOUL.md: one-shot, recurring, organized views (active/recurring/completed), conflict detection, disable/enable.

**Result**: Reminders fire at exact scheduled time via OpenClaw native cron. Tested and confirmed working.

### 8. Linked GWS skills to Master Control and Mr Wolf
Both agents had empty `skills/` directories — couldn't run GWS commands despite having Drive access documented.

Copied from Kai's workspace:
- `gws-shared` (auth layer)
- `gws-drive` (file operations)
- `gws-drive-upload` (upload files)
- `gws-docs` (read documents)

Added PDF + OCR workflow and Drive commands to both SOUL.md files.

**Result**: All three primary agents can now read, OCR, upload PDFs to their respective Drive folders.

### 9. Fixed Google Calendar syntax for Kai
Kai was stuck in a loop trying to create calendar events with `calendarId` in the wrong place.

**Root cause**: `calendarId` must go in `--params`, NOT in `--json` body.

Correct syntax:
```bash
gws calendar events insert \
  --params '{"calendarId":"kos.domus@gmail.com"}' \
  --json '{"summary":"...","attendees":[...]}'
```

Also added explicit rule: to share events with Katia, add her as attendee (`kati.zamboni@gmail.com`). Kai can't write to her calendar directly (different Google account).

**Result**: Calendar events with attendees working correctly.

### 10. Cleaned up and synced family contacts
Kai had added 9 contacts to its workspace `master_contacts.json` but the FamilyOS source copy was empty.

Cleaned up:
- Removed placeholder entry
- Fixed IDs to follow `ct_lastname_firstname` convention
- Fixed names (alias in notes, real name in `name` field)
- Synced to `~/job-desk/family-os/03_contacts/master_contacts.json`

8 contacts indexed: idraulico, elettricista, giardiniere, assicurazione, 2 meccanici, commercialista, pediatra.

Also added rule to Kai's SOUL.md: WhatsApp group is private, can share phone numbers and contacts freely when requested.

**Result**: Contacts properly indexed and synced.

### 11. Set up Google Meet recording pipeline
Created `Meet_Recordings/` folders on Drive for each agent:

| Agent | Drive Path | Folder ID |
|-------|-----------|-----------|
| Kai | Family/Meet_Recordings/ | DRIVE_ID_FAMILY_MEET |
| Master Control | Work/MasterControl/Meet_Recordings/ | DRIVE_ID_WORK_MEET |
| Mr Wolf | WIP/Meet_Recordings/ | DRIVE_ID_WIP_MEET |

Pipeline: user records Meet call → recording saves to Drive → user says "trascrivi l'ultimo meeting" → agent finds recording → moves to agent's folder → downloads → AssemblyAI pipeline → summary.

Added workflow to all three agent SOUL.md files with correct GWS commands for finding, moving, downloading recordings.

**Result**: Full Meet → transcript → summary pipeline for all agents.

### 12. Changed Kai's model to Gemini 3.1 Pro
Updated `openclaw.json` agent config:
- Primary: `google-gemini-cli/gemini-3.1-pro-preview`
- Fallbacks: `gemini-3-flash` → `gpt-5.4-mini` → `claude-cli/opus-4.6`

**Result**: Kai running on Gemini 3.1 Pro.

## Configuration Changes
- `~/job-desk/family-os/CLAUDE.md` → removed, replaced by `SOUL.md`
- `~/job-desk/family-os/SOUL.md` — new, full Kai agent config (OpenClaw native)
- `~/job-desk/family-os/11_decisions/` — new module (folders + schema)
- `~/job-desk/family-os/12_projects/` — new module (folders + schema)
- `~/job-desk/family-os/99_kai_system/schemas/entity_graph.json` — added decision + project entities
- `~/job-desk/family-os/99_kai_system/schemas/decision.schema.json` — new
- `~/job-desk/family-os/99_kai_system/schemas/project.schema.json` — new
- `~/job-desk/family-os/99_kai_system/scripts/audio-pipeline/` — new (7 Python files + shell wrapper)
- `~/job-desk/family-os/03_contacts/master_contacts.json` — populated with 8 contacts
- `~/.openclaw/workspace-family/SOUL.md` — reminders, calendar, Meet, contacts, privacy rules
- `~/.openclaw/workspace-orchestrator/SOUL.md` — Drive, PDF, audio, YouTube, Meet
- `~/.openclaw/workspace-wip/SOUL.md` — Drive, PDF, audio, YouTube, Meet
- `~/.openclaw/workspace-cos/SOUL.md` — YouTube
- `~/.openclaw/SOUL.md` — YouTube
- `~/.openclaw/workspace-orchestrator/skills/` — added gws-shared, gws-drive, gws-drive-upload, gws-docs
- `~/.openclaw/workspace-wip/skills/` — added gws-shared, gws-drive, gws-drive-upload, gws-docs
- `~/.openclaw/op-env.sh` — added ASSEMBLYAI_API_KEY
- `~/.openclaw/openclaw.json` — updated Kai model to gemini-3.1-pro-preview, OpenClaw 2026.4.2
- `~/.venvs/audio-pipeline/` — new Python venv with assemblyai SDK
- System packages: `ocrmypdf`, `tesseract-ocr-ita` installed via apt
- `youtube-transcript-api` installed via pipx

## Key Discoveries
- **AssemblyAI SDK outdated**: The Python SDK (v0.59.0) still uses deprecated `speech_model` (singular) while the API requires `speech_models` (plural list). Bypassed by using REST API directly via `urllib`.
- **Whisper large-v3 too heavy**: On the mini PC, transcribing a 5.5-min file took 10+ minutes at 46% CPU. AssemblyAI does it in 37 seconds with zero local CPU.
- **op-env.sh doesn't persist in exec**: When Kai runs `exec openclaw cron add`, the shell has no env vars. Must wrap in `bash -c 'source op-env.sh && ...'` with explicit PATH.
- **GWS calendarId placement**: Must go in `--params`, NOT in `--json` body. The API returns "Unknown property" if calendarId is in the request body.
- **OpenClaw cron --at is the reminder mechanism**: Native one-shot cron jobs fire at exact time and auto-delete. No polling file needed. This is how reminders were working before they broke.
- **Gateway must restart to load new cron jobs**: Jobs added directly to `jobs.json` aren't picked up until gateway restart.
- **Master Control and Mr Wolf had empty skills/**: Despite having Drive folder IDs documented in MEMORY.md, they couldn't actually run GWS commands because no skills were symlinked.
- **Meet recordings land in My Drive/Meet Recordings/**: Can't change the destination from Meet itself, but agents can move files to the correct folder post-recording.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| AssemblyAI `speech_models must be non-empty list` | SDK uses deprecated `speech_model` singular | Bypassed SDK, used REST API directly with `speech_models: ["universal-3-pro"]` |
| Whisper stuck on 5.5-min file (10+ min) | large-v3 model too heavy for mini PC CPU | Lowered Whisper threshold to 2 min, use AssemblyAI for everything longer |
| `openclaw cron add` → GatewaySecretRefUnavailableError | Env vars not loaded in exec shell | Wrapped in `bash -c 'export PATH=... && source op-env.sh && openclaw ...'` |
| 1Password session expired mid-pipeline | API key fetched per-chunk, session timed out | Cached key in memory after first read + added to op-env.sh for auto-load |
| Kai stuck on calendar event creation | `calendarId` in `--json` instead of `--params` | Fixed SOUL.md with correct syntax and tested |
| Reminders not firing | Kai writing to reminders.json but no cron runner | Switched to native `openclaw cron add --at` for exact-time delivery |
| Master Control / Mr Wolf can't run GWS | Empty skills/ directories | Copied gws-shared, gws-drive, gws-drive-upload, gws-docs from Kai |
| OpenClaw gateway running old 2026.4.1 code after npm update | Gateway process not restarted | `pkill openclaw && ~/.openclaw/start-gateway.sh` |

## Final State
- **FamilyOS**: 13 modules (11 original + decisions + projects), 9 schemas, entity graph with 7 entities
- **Audio pipeline**: AssemblyAI-powered, handles 30+ min files in <5 min, per-agent output routing
- **Reminders**: Fixed, using native OpenClaw cron `--at` jobs, fire at exact time
- **All agents**: YouTube transcripts, PDF/OCR, Drive upload, Meet recording pipeline
- **OpenClaw**: Updated to 2026.4.2 with security patches
- **Kai model**: Gemini 3.1 Pro (primary), with fallbacks to Flash, GPT, Claude
- **Contacts**: 8 useful contacts indexed in FamilyOS
- **Meet Recordings**: Dedicated Drive folders per agent (Family, Work, WIP)

## Open Questions
- ETL web research architecture: researched multi-layer scraping/crawling pipeline (API reverse engineering + headless browser). Saved as project memory, implementation pending.
- Should Meet recording processing be automatic (cron/webhook) or on-demand only?
- AssemblyAI free tier usage tracking — how much of the ~185 hours has been consumed?
- Firecrawl and xAI integration planned for future (new plugin format in 2026.4.2 ready)
- 1Password service account token setup for headless op access (currently requires manual signin or biometric)
