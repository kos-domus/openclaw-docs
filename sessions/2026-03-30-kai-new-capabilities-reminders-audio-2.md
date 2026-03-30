---
title: "Kai new capabilities spec: Smart Reminders and Audio Intelligence"
date: "2026-03-30"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "automation", "configuration", "scheduling", "skills"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
Define and save implementation specs for two major new Kai capabilities: Smart Reminder System and Audio Intelligence pipeline.

## Context
- Kai is the family AI agent (OpenClaw) operating in WhatsApp group "Family stuff"
- Kai already has WhatsApp access via OpenClaw Baileys integration — no Twilio needed
- FamilyOS data layer is on Google Drive under My Drive/Family/
- Kai's task specs are stored in 99_Kai_System/Tasks/ on Drive
- Ubuntu Mini PC is the always-on server with full Linux tooling (ffmpeg, pip, etc.)

## Steps Taken

### 1. Defined Smart Reminder System spec
Full feature spec covering:
- **3 reminder types**: Permanent/recurring, one-shot, intelligent/adaptive
- **Configuration**: content, datetime, cadence, advance notice, priority, expiry, notes
- **WhatsApp delivery**: priority-calibrated formatting, interactive responses (done/snooze/repeat/delete)
- **Conflict detection**: Kai detects but never resolves conflicts autonomously — notifies and waits for user decision
- **Natural language management**: "Kai, mostrami tutti i miei promemoria attivi"
- **Edge cases**: retry on delivery failure, holiday shift, 5-min grouping rule, ambiguous time expressions

Key design decision: **Twilio is NOT needed** — Kai already has WhatsApp access via OpenClaw's Baileys channel integration.

**Result**: Spec saved to `~/job-desk/family-os/99_kai_system/tasks/smart-reminder-system.md` and uploaded to Drive.

### 2. Defined Audio Intelligence spec
Full pipeline spec covering:
- **6-step pipeline**: Pre-processing → Transcription → Summary → Structured output → Translation → Optional enrichments
- **Input**: mp3, mp4, m4a, wav, ogg, webm, aac, flac + video tracks + live recording
- **Stack**: Whisper (local or API), pyannote.audio (diarization), ffmpeg (pre-processing), Claude/OpenAI API (analysis)
- **Output per session**: index, raw transcript, clean transcript, summary, outline, action items, key insights, open questions, named entities, translations
- **Optional enrichments**: sentiment analysis, glossary, flashcards, mind map, meeting minutes, content repurposing, comparative analysis
- **Privacy**: Whisper can run fully local, explicit confirmation required for sensitive content

Future integration options for iPhone access:
- Recommended: Obsidian Sync (consistent with .md architecture)
- Alternative: WhatsApp delivery (immediate, zero setup)
- Optional: DayOne via SSH to Mac (fragile, non-critical only)

**Result**: Spec saved to `~/job-desk/family-os/99_kai_system/tasks/audio-intelligence.md` and uploaded to Drive.

## Configuration Changes
- `~/job-desk/family-os/99_kai_system/tasks/smart-reminder-system.md` — NEW: full reminder system spec
- `~/job-desk/family-os/99_kai_system/tasks/audio-intelligence.md` — NEW: full audio pipeline spec
- Both uploaded to Google Drive under `Family/99_Kai_System/Tasks/`

## Key Discoveries
- **Twilio is unnecessary** — OpenClaw's existing Baileys WhatsApp integration already gives Kai message delivery capability, eliminating the need for Twilio API and its costs
- **Whisper runs locally on Ubuntu** — Full audio transcription without sending data to external servers, important for privacy-sensitive family content
- **Conflict detection vs resolution** — Critical design principle: Kai detects and notifies conflicts but never resolves them autonomously. The user always decides.
- **DayOne integration is architecturally fragile** — Requires Mac as SSH bridge (not guaranteed h24). Obsidian Sync is the cleaner alternative for iPhone-accessible archives

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| N/A — this was a spec/planning session | — | — |

## Final State
- Two comprehensive task specs saved for Kai (local + Google Drive)
- Session saved for Kos to elaborate into docs
- Implementation ready to begin in future sessions

## Open Questions
- How to handle reminder persistence across Kai agent restarts (Drive JSON read/write latency)?
- Should audio files be stored on Drive or only locally on the Mini PC (storage vs privacy tradeoff)?
- What's the max audio duration Whisper can handle locally on the Mini PC hardware?
- Should Kai support voice message input directly from WhatsApp (receive audio → auto-transcribe)?
- Obsidian Sync vs other options for iPhone access — needs evaluation session
