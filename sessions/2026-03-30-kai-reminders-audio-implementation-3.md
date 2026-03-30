---
title: "Kai Smart Reminders and Audio Intelligence: full implementation"
date: "2026-03-30"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "automation", "cron", "scheduling", "configuration", "setup", "skills"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
Build (not spec) two new capabilities for Kai: a Smart Reminder System with WhatsApp delivery and an Audio Intelligence pipeline with local Whisper transcription.

## Context
- Feature specs were drafted earlier in the session but the user requested actual implementation, not documentation
- Kai is an OpenClaw agent — instructions go in workspace bootstrap files (SOUL.md, TOOLS.md, HEARTBEAT.md, MEMORY.md), NOT in CLAUDE.md
- Kai already has WhatsApp access via OpenClaw Baileys integration — no Twilio needed
- FamilyOS data layer is on Google Drive under My Drive/Family/
- Ubuntu Mini PC is the always-on server

## Steps Taken

### 1. Built Reminder Data Layer on Drive
Created reminder schema (`reminder.schema.json`) with fields: reminder_id, content, datetime, cadence, priority, status, recurrence_rule, snooze_until, delivery_history, source (explicit/inferred).

Created three data files on Drive under `99_Kai_System/Reminders/`:
- `reminders.json` — active reminder registry (append-only)
- `reminders_archive.json` — completed/cancelled reminders
- `conflicts_log.json` — conflict detection and resolution log

Uploaded schema to Drive Schemas folder.

**Result**: All files created with stable Drive IDs, ready for Kai to read/write.

### 2. Created Reminder Checker Cron Job
Added OpenClaw cron job for Kai:
- Schedule: every 5 minutes (`*/5 * * * *`)
- Agent: family (Kai)
- Delivery: WhatsApp group
- Logic: download reminders.json, find due reminders (active + datetime passed), deliver formatted notifications, update delivery_history, re-upload

Also created `check-reminders.sh` helper script that downloads and filters due reminders via Python.

**Result**: Cron job registered (`49f50a09-...`), runs every 5 minutes.

### 3. Installed Audio Intelligence Stack
Installed on Ubuntu:
- **ffmpeg** v6.1.1 via apt — audio pre-processing, conversion, noise reduction
- **OpenAI Whisper** v20250625 in Python venv at `/home/kos/.venvs/whisper/` — local transcription

Created `process-audio.sh` pipeline script:
1. Pre-process with ffmpeg (16kHz mono WAV, noise reduction)
2. Analyze audio duration
3. Transcribe with Whisper (model: small, language: auto)
4. Output: raw transcript, timestamped VTT/SRT, JSON data, index.md

Created `Audio_Processing/` folder on Drive for output storage.

**Result**: Full pipeline tested — ffmpeg and Whisper both verified working.

### 4. Updated Kai's Bootstrap Files (NOT CLAUDE.md)
Critical lesson: OpenClaw agents read workspace bootstrap .md files, not CLAUDE.md.

**SOUL.md** — Added two full sections:
- Smart Reminder System: create/deliver/snooze/conflict detection workflows, formatting per priority, grouping rule, conflict notification format
- Audio Intelligence: pipeline steps, supported formats, output types, WhatsApp voice message handling, privacy rules

**TOOLS.md** — Added:
- exec access to audio pipeline script
- Whisper venv path
- ffmpeg availability

**HEARTBEAT.md** — Added section 4: Reminder check (download reminders.json, deliver due reminders, update delivery_history, calculate next occurrence for recurring)

**MEMORY.md** — Added:
- Reminder file IDs (reminders.json, archive, conflicts_log) to the file ID table
- Reminders/ and Audio_Processing/ folder IDs to the Drive tree
- Full reminder JSON schema with all fields documented
- Reminder workflow (download → modify → re-upload pattern)

**Result**: All instructions in Kai's actual operational files. Kai will read these on every session start.

### 5. Cleaned Up Spec Files
Removed the `Tasks/` folder from Drive that contained spec files (smart-reminder-system.md, audio-intelligence.md). These were replaced by actual working infrastructure and operational instructions in bootstrap files.

**Result**: Drive clean — only working data files, no spec documents.

## Configuration Changes
- `~/.openclaw/workspace-family/SOUL.md` — Added Smart Reminder System + Audio Intelligence sections
- `~/.openclaw/workspace-family/TOOLS.md` — Added exec for audio pipeline, Whisper venv, ffmpeg
- `~/.openclaw/workspace-family/HEARTBEAT.md` — Added reminder check as section 4
- `~/.openclaw/workspace-family/MEMORY.md` — Added reminder file IDs, Drive folder IDs, reminder schema, reminder workflow
- `~/.openclaw/cron/jobs.json` — Added "Reminder checker" job (every 5 min, Kai agent, WhatsApp delivery)
- Installed: ffmpeg (apt), openai-whisper (pip in venv)
- Created: `process-audio.sh`, `check-reminders.sh` scripts
- Drive: Created Reminders/ folder + 3 JSON files, Audio_Processing/ folder, reminder schema

## Key Discoveries
- **CLAUDE.md is NOT read by OpenClaw agents** — This was the third time this lesson came up. OpenClaw agents use SOUL.md/TOOLS.md/AGENTS.md/HEARTBEAT.md/MEMORY.md. CLAUDE.md is only for Claude Code CLI. Must stop defaulting to it.
- **Whisper needs a Python venv on Ubuntu 24.04** — `pip install openai-whisper` fails with PEP 668 externally-managed-environment error. Solved with `python3 -m venv` then install inside.
- **Twilio is unnecessary for Kai's WhatsApp delivery** — OpenClaw's existing Baileys integration already provides the channel. One less dependency, zero cost.
- **Cron jobs for reminders should be lightweight** — The 5-minute checker only reads a JSON file and checks timestamps. If no reminders are due, Kai does nothing (no token cost). Only triggers full agent response when a reminder is actually due.
- **Spec files should not live on production Drive** — Implementation artifacts (schemas, data files, scripts) replace specs. Specs were deleted after implementation.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| `pip install openai-whisper` → PEP 668 error | Ubuntu 24.04 blocks system-wide pip installs | Created venv at `~/.venvs/whisper/` |
| `gws: command not found` in Bash tool | PATH not including npm-global/bin | Explicit `export PATH="$PATH:/home/kos/.npm-global/bin"` |
| Instructions put in CLAUDE.md instead of bootstrap files | Force of habit from Claude Code conventions | User corrected — moved all instructions to SOUL.md, TOOLS.md, HEARTBEAT.md, MEMORY.md |

## Final State
- **Smart Reminder System**: Fully built — schema, Drive storage, cron checker (every 5 min), operational logic in Kai's SOUL.md, file IDs in MEMORY.md
- **Audio Intelligence**: Fully built — ffmpeg + Whisper installed, pipeline script ready, Drive folder created, operational logic in Kai's SOUL.md
- **Kai bootstrap files**: All four updated (SOUL, TOOLS, HEARTBEAT, MEMORY) with complete operational instructions
- **Drive**: Clean — only working data files, spec folder removed

## Open Questions
- Should the reminder cron job run every 5 minutes or is that too frequent (token cost)?
- Whisper "small" model is the default — should we use "medium" or "large" for better accuracy on Italian audio?
- pyannote.audio (speaker diarization) was not installed — add it when multi-speaker transcription is needed?
- How to test the reminder system end-to-end? Create a test reminder for 5 minutes from now?
- Voice messages from WhatsApp: does Kai's OpenClaw channel forward audio files, or only text?
