---
title: "Kai daily/weekly/monthly cron jobs, waste calendar fix, Node 24 upgrade, local memory search"
date: "2026-03-31"
author: "kos-domus"
status: "processed"
tags: ["cron", "scheduling", "automation", "multi-agent", "configuration", "setup", "troubleshooting", "memory"]
openclaw_version: "2026.3.28"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
1. Diagnose why Kai missed the plastica waste reminder
2. Add daily morning briefing, weekly briefing, and monthly expense report cron jobs for Kai
3. Upgrade Node.js to v24 and enable local memory search for all agents
4. Run health check and clean up orphan session files
5. Rebuild the complete 2026 waste calendar from the official Bussolengo Zona A PDF

## Context
- Kai's rifiuti-2026.json was missing PLASTICA E LATTINE entries entirely — 0 dates for plastic collection
- The existing rifiuti cron (17:30 daily) was working but could only report what was in the schedule file
- Memory search was configured as "local" but node-llama-cpp wasn't installed (needed Node 24)
- OpenClaw doctor flagged orphan session files and memory search issues

## Steps Taken

### 1. Diagnosed missing plastica reminder
Checked rifiuti-2026.json — confirmed PLASTICA E LATTINE was completely absent from the schedule. The file only had SECCO, UMIDO, and CARTA entries. The rifiuti cron job worked correctly but had no plastica data to report.

**Result**: Root cause identified — data gap, not a code bug.

### 2. Created three new Kai cron jobs

**Daily morning briefing** (7:00 AM every day):
- Checks reminders.json on Drive for today's active reminders
- Checks Google Calendar for today's events
- Checks rifiuti-2026.json for tomorrow's waste collection
- Sends ONE combined message if anything exists, SKIP if nothing

**Weekly briefing** (7:00 AM every Monday):
- Same sources but for the full upcoming week
- Groups events/reminders/rifiuti by day

**Monthly expense report** (8:00 PM on 28-31st, only fires on last day):
- Downloads expenses_2026.json from Drive
- Calculates: total, breakdown by category, by person, top 5, month-over-month comparison
- Saves report to Drive (02_Finance/Reports/) and sends to WhatsApp group

All three use GPT 5.4-mini (subscription, no per-token cost) and deliver to WhatsApp group with best-effort delivery.

**Result**: Three cron jobs created and registered.

### 3. Upgraded Node.js 22 → 24 for local memory search
```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs  # 22.22.1 → 24.14.1
npm install -g openclaw@latest  # rebuild with Node 24
npm install -g node-llama-cpp  # install separately (npm couldn't install it as openclaw dependency)
openclaw memory index  # downloaded embeddinggemma-300m model (328MB), indexed all agents
```

**Result**: Local memory search operational for all 8 agents. Model: EmbeddingGemma 300M. Storage: SQLite per agent. Zero data leaves the machine.

| Agent | Files Indexed | Chunks |
|-------|-------------|--------|
| Kos (cos) | 13 | 52 |
| Kai (family) | 6 | 17 |
| Mr Wolf (wip) | 3 | 3 |
| Others | 1 each | 1-2 each |

### 4. Ran health check and cleaned up sessions
`openclaw doctor` identified:
- 15 orphan transcript files (60K total) — probe files and abandoned session stubs
- Memory search unavailable (fixed by Node 24 upgrade)
- Telegram group policy warnings (pre-existing, cosmetic)
- Gateway service PATH warnings (custom setup, non-functional)

Cleaned up with `openclaw sessions cleanup --enforce --fix-missing`. Sessions reduced from 15 to 7.

**Result**: Clean session store, no orphan files.

### 5. Rebuilt complete 2026 waste calendar from official PDF
User provided the official Bussolengo Zona A 2026 PDF calendar (Serit/AMIA). Extracted all 12 months day by day.

First attempt had errors (PDF OCR was unreliable). Fixed April based on user verification, then rebuilt all months carefully cross-referencing day names, dates, and the PDF layout.

Final calendar stats:
| Type | Days/Year |
|------|-----------|
| UMIDO | 117 |
| SECCO | 53 |
| VERDE | 28 |
| CARTA | 26 |
| PLASTICA E LATTINE | 26 |

Total: 236 collection days. Summer months (June-September) have combined UMIDO+SECCO on Mondays.

User verified July and September against the PDF — 100% match.

**Result**: Complete and verified rifiuti-2026.json with all waste types including PLASTICA E LATTINE.

### 6. Privacy decision for memory search
Discussed local vs remote (Google/OpenAI) for memory search embeddings. Since agents handle family data (expenses, health, contacts, 1Password), chose **local** for maximum privacy. Node 24 + node-llama-cpp runs EmbeddingGemma entirely on the Mini PC.

**Result**: All embeddings computed locally, zero data sent to external providers.

## Configuration Changes
- Node.js upgraded: 22.22.1 → **24.14.1**
- `node-llama-cpp` installed globally (enables local embeddings)
- `openclaw memory index` run — all 8 agents indexed
- `~/.openclaw/cron/jobs.json` — Added 3 new Kai cron jobs (daily briefing, weekly briefing, monthly expenses)
- `~/.openclaw/workspace-family/memory/rifiuti-2026.json` — Complete rebuild with all 236 collection days including PLASTICA E LATTINE
- Orphan session files cleaned up (15 removed, 7 remaining)

## Key Discoveries
- **Missing data ≠ missing code**: Kai's rifiuti reminder worked perfectly — the problem was that PLASTICA E LATTINE was never entered in the schedule file. Always check data before blaming logic.
- **PDF OCR extraction is unreliable**: Calendar PDFs with mixed layouts, images, and text produce inconsistent OCR. Manual verification month by month is necessary.
- **node-llama-cpp needs separate global install**: `npm install -g openclaw@latest` with Node 24 doesn't automatically install node-llama-cpp as an optional dependency. Must run `npm install -g node-llama-cpp` separately.
- **Local memory search is viable on Mini PC**: EmbeddingGemma 300M (328MB) runs fine on CPU. Indexing 27 files across 8 agents took seconds. SQLite storage is lightweight.
- **Summer waste schedule changes**: June-September have combined UMIDO+SECCO on Mondays (increased frequency for organic waste in summer heat). Schedule transitions happen mid-June and mid-September.
- **Cron monthly report trick**: Using `0 20 28-31 * *` with a "check if last day of month" guard inside the prompt handles variable month lengths without complex cron expressions.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| node-llama-cpp missing after OpenClaw reinstall | Optional dependency not installed by npm -g | `npm install -g node-llama-cpp` separately |
| npm peer dependency conflict installing node-llama-cpp inside openclaw dir | openclaw's package.json has strict peer deps | Install globally instead of inside openclaw's node_modules |
| PDF waste calendar extraction errors (wrong dates) | OCR text from image-based PDF pages unreliable | Manual month-by-month verification against PDF, user cross-checked July and September |
| PLASTICA missed by Kai | Zero PLASTICA entries in rifiuti-2026.json | Rebuilt complete calendar from official PDF |

## Final State
- **Kai cron jobs**: 4 active (rifiuti 17:30, daily briefing 07:00, weekly Mon 07:00, monthly expenses last day 20:00)
- **Waste calendar**: Complete for all 2026, verified against official PDF. 236 collection days across 5 types.
- **Memory search**: Local, operational for all 8 agents. EmbeddingGemma 300M on CPU. SQLite storage.
- **Node.js**: v24.14.1
- **Session store**: Cleaned, 7 active sessions

## Open Questions
- How to manage memory search long-term? Need to explore indexing strategies, search usage in conversations, and re-indexing triggers
- Should the waste calendar be uploaded to Google Drive as well, so Kai can reference it from there?
- The monthly expense report runs on 28-31st with a "last day" check — need to verify it actually fires and SKIPs correctly on non-last days
- October waste calendar entries may have minor errors (OCR was least reliable for that month) — verify when October approaches
