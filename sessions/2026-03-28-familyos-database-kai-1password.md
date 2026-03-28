---
title: "FamilyOS semantic database on Drive, Kai learning layer, security audit, 1Password integration"
date: "2026-03-28"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "security", "automation", "setup", "memory", "claude-md"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Build a semantic, AI-query-friendly family database (FamilyOS) for Kai (the family WhatsApp bot agent), deploy it on Google Drive, add a memory/learning layer so Kai can learn across conversations, run a full security audit of the OpenClaw environment, and begin integrating 1Password as the unified secrets provider.

## Context
- Kai is a dedicated OpenClaw agent managing the WhatsApp group "Family stuff" (Kos, Katia, and bot)
- Kai is separate from Kos (the documentation agent) — different agent, different purpose
- Family: Kos (father), Katia (mother), Mar (daughter), Chris (son, expected June 2026)
- Kai has GWS (Google Workspace CLI) access with active OAuth
- Google Drive already structured with 3 top-level folders: Family, WIP, Work
- openclaw-docs serves as the home base project; FamilyOS is the first case study

## Steps Taken

### 1. Designed FamilyOS semantic data layer
Created a full database structure following these design principles:
- **JSON-first**: PDF for archival, JSON for intelligence
- **Entity linking**: Every record references person_id, category, date
- **Single source of truth**: master_contacts.json is THE contacts file, not folder copies
- **Consistent naming**: YYYY-MM-DD_type_description.ext for timestamped files
- **Query-friendly**: Entity graph defines all relationships and traversal patterns

Structure (11 modules):
```
01_Family_Members/  → profile.json per person (central identity nodes)
02_Finance/         → expenses (raw + processed JSON), reports, views by person/category
03_Contacts/        → master_contacts.json as single source of truth + category views
04_Schedule/        → events, recurring, school calendar, holidays
05_Activities/      → sports, music, education, courses (hub linking costs/contacts/events)
06_Documents/       → legal, insurance, identity, medical, school, housing
07_Health/          → per-person: visits, vaccinations, reports
08_Family_Knowledge → preferences, routines, decisions, notes
09_Assets/          → subscriptions, devices
10_Travel/          → trips, docs, expenses
99_Kai_System/      → schemas, entity graph, Kai config, memory, prompts, logs
```

**Result**: Local project created at ~/job-desk/family-os/ with full structure, 5 JSON schemas, entity graph, profiles, and CLAUDE.md for Kai.

### 2. Created 5 JSON schemas for data validation
Schemas define every data type with consistent ID patterns and cross-references:
- `family_member.schema.json` — person_id as central node, links to everything
- `expense.schema.json` — exp_YYYYMMDD_NNN format, links to person_ids and activity_id
- `contact.schema.json` — ct_lastname_firstname format, links to related_to person_ids
- `event.schema.json` — evt_YYYYMMDD_shortname, links to person_ids, contact_ids, activity_id
- `activity.schema.json` — act_category_shortname, hub entity linking all others

**Result**: All schemas with JSON Schema draft 2020-12, full validation rules and relationship definitions.

### 3. Built entity graph for AI traversal
Created `entity_graph.json` defining:
- All entity relationships with cardinality (one_to_many, many_to_many)
- Source file patterns for each entity type
- Pre-built query patterns (SQL-like pseudo-queries) for common operations:
  - expenses_by_person_and_month
  - monthly_report_per_child
  - upcoming_conflicts
  - activity_total_cost

**Result**: Kai can traverse family_member ↔ expense ↔ activity ↔ contact ↔ event in any direction.

### 4. Deployed FamilyOS to Google Drive via GWS
Decision: Move from local git to Google Drive because:
- Kos and Katia can browse from any device (phone, browser)
- Kai accesses via GWS CLI (already configured with OAuth)
- Drive handles versioning natively

Used GWS CLI to create the full folder structure under `My Drive/Family/`:
```bash
# Create top-level folder
gws drive files create --json '{"name": "01_Family_Members", "mimeType": "application/vnd.google-apps.folder", "parents": ["DRIVE_FAMILY_ROOT_ID"]}'

# Upload JSON files
gws drive files create --json '{"name": "profile.json", "parents": ["FOLDER_ID"]}' \
  --upload "01_family_members/kos/profile.json" --upload-content-type "application/json"
```

Key folder IDs stored for Kai's reference:
- Family root: `DRIVE_FAMILY_ROOT_ID`
- 99_Kai_System: `DRIVE_KAI_SYSTEM_ID`
- Memory folder: `DRIVE_MEMORY_FOLDER_ID`

**Result**: 11 top-level folders, all subfolders, and 15 JSON seed files uploaded to Drive. GWS `--upload` requires running from within the project directory (relative paths only).

### 5. Formalized upstream sources as permanent references
Updated `docs/meta/upstream-version.yaml` with:
- `permanent: true` flag on all 3 sources (GitHub, npm, docs)
- Concrete `check_methods` with actual CLI commands:
  - `gh api repos/openclaw/openclaw/releases/latest`
  - `npm view openclaw version`
- Expanded key_pages with tracking descriptors (what each page monitors)
- Added MCP and hooks pages to monitoring list

**Result**: Upstream sync protocol now has machine-executable check commands, not just URLs.

### 6. Added memory/learning layer to Kai
Major gap identified: Kai had zero memory across conversations — purely reactive.

Created new schema + 3 memory files:
- `memory.schema.json` — 6 memory types: correction, preference, pattern, decision, observation, feedback. Includes confidence scoring (0.0-1.0) and supersession chains.
- `memories.json` — Append-only memory store, seeded with 3 initial memories
- `patterns.json` — Rolling baselines for adaptive anomaly detection (spending averages, activity frequency)
- `learned_preferences.json` — Per-person communication and behavior preferences

Updated Kai's CLAUDE.md with:
- Full memory system instructions (read-before-act, append-only, confidence scoring)
- Proactive intelligence rules (suggest before asked, anticipate needs, learn rhythms)
- Drive-native GWS workflows replacing local filesystem operations
- Scope boundary: Kai ONLY accesses My Drive/Family/, never WIP/ or Work/
- Key Drive folder IDs for cached reference

**Result**: Kai now has persistent memory, adaptive thresholds, and proactive learning capabilities.

### 7. Security audit of ~/.openclaw and ~/.claude
Comprehensive scan of both directories:

**~/.openclaw (700 permissions — correct)**:
- Contains: GitHub PAT, GCP service account key, WhatsApp auth (Baileys), Gmail app password, Groq API key, OpenRouter API key, device tokens, OAuth tokens
- All credential files at 600 permissions (correct)
- SOUL.md says "secrets: never in files, only env vars" but plaintext files exist (.secrets, .secrets-gcp.json, .env)

**~/.claude (775 permissions — FIXED to 700)**:
- Contains OAuth tokens (access + refresh) for Claude.ai
- Was world-readable — fixed with `chmod 700`

**Repo credential scan**:
- openclaw-docs: **Clean** — all token references are redacted examples (sk-ant-api03-...)
- family-os: **Clean** — zero sensitive data
- PII scan (phone numbers, emails): **Clean** in both repos

**Result**: ~/.claude permissions fixed. No credential leaks in repos. Plaintext secrets in ~/.openclaw identified for 1Password migration.

### 8. Began 1Password integration (in progress)
Chose **1Password Families** plan ($4.99/month, 5 members).

Vault structure designed:
| Vault | Contents | Access |
|-------|----------|--------|
| Family | WiFi, router, home alarm, school portals | Kos + Katia |
| Streaming | Netflix, Disney+, Spotify | Kos + Katia |
| Utilities | Electricity, gas, internet, insurance | Kos + Katia |
| Medical | Health portals, pharmacy, pediatrician | Kos + Katia |
| Finance | Banking, tax portal, INPS, SPID | Kos + Katia |
| Tech | OpenClaw API keys, GCP, GitHub, Telegram bots | Only Kos |

Installed `op` CLI v2.33.1 via apt:
```bash
curl -sS https://downloads.1password.com/linux/keys/1password.asc | sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/amd64 stable main" | sudo tee /etc/apt/sources.list.d/1password.list
sudo apt install 1password-cli
```

Account configured (`op account add`), vaults created by user, Katia invited.

**Blocker**: Desktop app installed via snap but CLI installed via apt — snap sandbox prevents socket communication (`~/.1password/agent.sock` not created). Fix identified: remove snap version, install via apt.

**Result**: CLI installed, account linked, vaults created. Blocked on snap vs apt desktop app conflict — needs `sudo snap remove 1password && sudo apt install 1password`.

## Configuration Changes
- `docs/meta/upstream-version.yaml` — Added permanent flag, check_methods, expanded key_pages
- `~/.claude` permissions — Changed from 775 to 700
- `~/job-desk/family-os/CLAUDE.md` — Full rewrite with Drive workflows, GWS commands, memory system, scope boundaries
- `~/job-desk/family-os/99_kai_system/` — Added memory/ folder with 3 JSON files + memory schema
- 1Password apt repo + debsig policy added to system
- 1Password CLI v2.33.1 installed

## Key Discoveries
- **GWS `--upload` requires relative paths**: Cannot use absolute paths outside the current working directory — must `cd` into the project first
- **Snap vs apt 1Password conflict**: Desktop app from snap sandboxes the agent socket, making CLI integration impossible. Both must come from the same source (apt)
- **Entity graph > folder hierarchy**: For AI agents, the relational graph (entity_graph.json) matters more than the directory structure. The same data can be queried by person, by category, by date, or by activity — the folders are just a human convenience layer
- **Memory types matter**: Corrections and preferences need different handling than observations. Confidence scoring lets the agent weigh explicit statements (1.0) differently from inferences (0.5)
- **~/.claude defaulted to 775**: Claude Code's config directory was world-readable out of the box — security risk on multi-user systems

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| GWS `--upload` path outside current directory | GWS validates upload paths are relative to cwd | Run GWS commands from within the project directory |
| `op whoami` → "no active session" | Desktop app installed via snap, CLI via apt — socket isolation | Remove snap version, install desktop app via apt |
| `op signin` accepts password silently but no session | Same snap/apt socket conflict | Pending: `sudo snap remove 1password && sudo apt install 1password` |
| ~/.claude at 775 permissions | Default installation behavior | `chmod 700 ~/.claude` |

## Final State
- **FamilyOS**: Fully deployed on Google Drive with 11 modules, 6 schemas, entity graph, 4 profiles, memory layer
- **Kai CLAUDE.md**: Complete with Drive-native workflows, GWS reference, memory/learning system, proactive intelligence rules
- **Upstream sources**: Formalized as permanent with executable check commands
- **Security**: ~/.claude fixed, both repos clean, plaintext secrets in ~/.openclaw identified for migration
- **1Password**: CLI installed, account linked, 6 vaults created, Katia invited. Blocked on snap removal — next session resumes here

## Open Questions
- How to handle Kai reading passwords from 1Password in WhatsApp group (privacy: never send passwords in plaintext to group chat?)
- Should Kai have read-only access to shared vaults, or also write (to save new passwords)?
- What's the session token refresh strategy for `op` on an always-on server?
- Cost of running Kai's memory reads (downloading memories.json from Drive) at every conversation start — will it hit Drive API limits?
- How to handle Chris's profile activation when born (manual trigger or Kai detects birth announcement?)
