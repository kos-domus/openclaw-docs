---
title: "Fleet specialization, daily cron pilot, Obsidian vault sync, and Local REST API MCP integration"
date: "2026-04-27"
author: "kos-domus"
status: "processed"
tags: ["agents", "fleet", "soul-md", "bootstrap", "cron", "drive", "obsidian", "syncthing", "autossh", "mcp", "tunnel", "reverse-tunnel", "tailscale"]
openclaw_version: "2026.4.24"
environment:
  os: "Ubuntu 25.10 (kernel 6.17.0-22-generic) on mini PC, macOS 15 on MacBook Pro"
  ide: "VS Code with Claude Code extension (Remote SSH from Mac to mini PC)"
  model: "claude-opus-4-7-1m"
---

## Objective

Three separate but related goals:

1. **Operationalize the multi-agent fleet** by giving each specialist a real domain identity (instead of identical boilerplate SOUL.md), per-agent Drive output folders with permission boundaries, and a pilot of cron-driven tasks coordinated by Master Control.
2. **Set up an Obsidian vault** as the personal "diario operativo" for Rakki, replicated across mini PC ↔ Mac via Syncthing and to the cloud via Obsidian Sync, with Master Control writing into it via cron.
3. **Wire the Obsidian Local REST API** into Claude Code as an MCP tool, so future sessions can semantically query the vault (backlinks, frontmatter, tags) instead of `Read`/`Grep`.

## Context

- The 4 specialist agents (`frontend-specialist`, `backend-expert`, `cso`, `orchestration-architect`) had identical generic `SOUL.md` boilerplate — they were not actually specialists, just personality stubs. The Master Control routing table existed but had no real downstream substance.
- The Drive structure under `Work/` (folders for MasterControl, Frontend, Backend, CSO, OrchArch, Job Projects, Shared) had been pre-created by Rakki, but the agents had no way to address those folders concretely (no folder IDs, no per-agent permission map).
- Obsidian was a candidate "knowledge plane" sitting on top of the existing markdown ecosystem (openclaw-docs sessions, project DOSSIER, agent memories) but Rakki's mini PC is headless and Obsidian is GUI-only — so a sync layer between mini PC (where MC writes) and Mac (where Obsidian runs) was needed.

## Steps Taken

### 1. Drive folder registry + per-agent permission table

Created `~/.openclaw/memory/FLEET-DRIVE.md` as the canonical reference for every Drive folder ID + the read/write authority of each agent. Discovered all 7 folders already existed under `Work/`; just had to record their IDs and codify the permission rules:

| Agent | Owns (write) | Reads | Local FS authority |
|---|---|---|---|
| Master Control (`orchestrator`) | entire `Work/` + `MasterControl/` | all subfolders | r+w on `/home/kos/job-desk/` |
| Each specialist | their own `Work/<Name>/` | `Job Projects/`, `Shared/`, peer folders (read) | read-only on local project FS |

Also defined the **output convention** every agent report follows: filename `YYYY-MM-DD-<task-slug>.md`, frontmatter with `agent`, `task`, `project`, `date`, `priority`, `effort`, and required body sections `Findings / Recommendations / Risk / References`.

**Result**: a single file (`FLEET-DRIVE.md`) is the source of truth for every folder ID, so each agent's SOUL just references it instead of hardcoding IDs.

### 2. Domain-specific SOUL.md for each specialist

Rewrote `SOUL.md` for the four specialists, replacing the generic "be helpful" boilerplate with:

- **Identity** specific to the role (Frontend Specialist, Backend Expert, CSO, Orchestration Architect)
- **Domain boundaries** (what's yours, what's NOT yours — including explicit deflection rules)
- **Principles** specific to the domain (e.g. CSO has "severity-graded findings" + "veto on Critical only"; OrchArch has "consistency check is your superpower")
- **Drive workspace** with the folder ID baked in (cross-referenced to `FLEET-DRIVE.md`)
- **Output convention** + upload command snippet
- **Task list** — concrete recurring tasks the cron may invoke
- **Default cross-cutting weekly task** — "for each active project, identify N inefficiencies in your domain"

Each SOUL ended up around 60-80 lines, intentionally specific. The boundary clauses are load-bearing — they prevent role drift when Master Control delegates ambiguous work.

### 3. Master Control SOUL update

Patched `~/.openclaw/workspace-orchestrator/SOUL.md`:

- Replaced the path-based Drive section with an explicit folder-ID table (per-agent r/w mapping)
- Added explicit local filesystem authority (`/home/kos/job-desk/` r+w, the rest of the agents read-only)
- Added a "Recurring duties" section documenting the Daily Work Briefing (06:30) + Evening Digest (19:30) cron contracts so MC remembers them at boot

### 4. BOOTSTRAP.md per agent

`openclaw status` reported `8 · no bootstrap files`. Discovered OpenClaw 2026.4.24 looks for these specific filenames in each workspace: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `TOOLS.md`, `USER.md`, `MEMORY.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`. Three workspaces had a custom `BOOT.md` (different filename, ignored by OpenClaw); five had nothing.

```bash
# For each workspace: rename existing BOOT.md → BOOTSTRAP.md, or create a template
for a in cos family wip orchestrator cso frontend-specialist backend-expert orchestration-architect; do
  ws=~/.openclaw/workspace-$a
  if [ -f "$ws/BOOT.md" ]; then
    cp "$ws/BOOT.md" "$ws/BOOTSTRAP.md"
  elif [ ! -f "$ws/BOOTSTRAP.md" ]; then
    # write minimal BOOTSTRAP.md template
    cat > "$ws/BOOTSTRAP.md" <<EOF
# BOOTSTRAP.md — <Agent Display Name>
... (read SOUL.md, IDENTITY.md, MEMORY.md; verify channel; ready)
EOF
  fi
done
```

After this, `openclaw status` reported `8 · 8 bootstrap files present`.

### 5. Cross-provider fallback chains applied per agent

Building on the 2026-04-26 doc work, applied the cross-provider rule concretely:

- Top tier (cos / family / wip / orchestrator): primary `openai-codex/gpt-5.4`, fallbacks Gemini 2.5 Flash + Claude Opus 4.6 + Codex 5.4-mini (4 distinct providers)
- Worker tier (cso / frontend / backend / orchestration-architect): primary `openai-codex/gpt-5.4-mini` with same cross-provider fallbacks

Used a Python script that edits `~/.openclaw/openclaw.json` while the gateway is **stopped** (the gateway rewrites the file from in-memory state on graceful restart, so direct edits while running get clobbered). Lesson learned during this session.

Also caught two invalid keys (`tools.allowedPaths` and `plugins.bonjour`) added during exploration that broke schema validation. Removed them; bonjour disable goes through `openclaw plugins disable bonjour` (which has its own correct config-rewrite path).

### 6. Removed `embeddedHarness: google-gemini-cli` from orchestrator

The Daily docs elaboration cron kept failing with `Unknown arguments: skip-trust, skipTrust` even after the model was changed away from Gemini. Root cause: `orchestrator` had `embeddedHarness.runtime: google-gemini-cli`, which means the agent runs *inside* a `gemini` CLI wrapper regardless of the underlying model. The OpenClaw 2026.4.24 source hardcodes `--skip-trust` flag in `dist/extensions/google/cli-backend.js` lines 26 and 33, but the current Gemini CLI (0.36.0) rejects that flag.

Workaround: remove `embeddedHarness` from orchestrator entirely, falling back to OpenClaw's default harness. The cron then ran successfully end-to-end with `openai-codex/gpt-5.4`.

### 7. Daily docs elaboration cron — three live bugs fixed

The existing `Daily docs elaboration` cron had been broken for 3 consecutive days with three different errors:

| Day | Error | Root cause |
|---|---|---|
| Apr 24 | `outside the allowed workspace... /home/kos/.openclaw/workspace-orchestration-architect` | Subagent sandbox didn't include `/home/kos/job-desk/openclaw-docs/` |
| Apr 25 | `All models failed: gemini-3.1-pro-preview capacity exhausted | gemini-2.5-flash capacity exhausted` | Mono-provider fallback chain hit Gemini saturation |
| Apr 27 morning | `Unknown arguments: skip-trust, skipTrust` | The `embeddedHarness` issue described in step 6 |

After step 6's fix + a model change to `openai-codex/gpt-5.4`, a manual `openclaw cron run` completed in 213s with status `ok` and successfully processed `2026-04-25-saas-delivery-receipts...md` (status flipped to `processed`, upstream-version.yaml refreshed, docs/index.yaml regenerated, changelog entry written).

### 8. Six pilot cron jobs registered

Created six new cron jobs as Phase B of the multi-agent operationalization:

| Cron | Schedule | What it does |
|---|---|---|
| MC: Daily Work Briefing | 06:30 daily | Enumerate projects + blocks, write to Drive `MasterControl/`, daily note in Obsidian vault, Telegram summary |
| MC: Evening Digest | 19:30 daily | Aggregate today's specialist reports, extract H/M priority, archive in `Shared/`, Telegram digest |
| Backend: DB Schema Review | Mon 04:00 | MC delegates to Backend Expert, target = SpesaBot |
| CSO: Secret & Credential Audit | Mon 04:30 | MC delegates to CSO, scope = all `~/job-desk/` projects |
| Frontend: UX Friction Audit | Wed 05:00 | MC delegates to Frontend Specialist, target = Spesify Mini App |
| OrchArch: System Design + Cross-Agent Consistency | Fri 05:00 | MC delegates to OrchArch — also reads Backend/Frontend/CSO reports of the week and flags conflicts |

All cron jobs target `--agent orchestrator` (so MC always coordinates and delegates), deliver to `telegram:877281564`, and use `--best-effort-deliver` so a Telegram failure doesn't fail the whole job.

### 9. Obsidian vault setup

Vault at `~/Obsidian-Personal/` with structure `Daily/`, `Decisions/`, `Inbox/`, `Templates/` (containing a Templater-compatible `Daily.md` template + a CLAUDE.md guide for Claude Code sessions opened in the vault). Created a sibling `~/job-desk/CLAUDE.md` so any project workspace under `job-desk/` knows about the vault as cross-project knowledge layer.

### 10. Syncthing P2P sync between mini PC and Mac

Installed `syncthing` v2.0.16 userspace on mini PC (no sudo, binary in `~/.local/bin`, systemd user service for autostart). Generated config, replaced default empty folder with vault folder (id `obsidian-personal`, path `/home/kos/Obsidian-Personal`).

On the Mac side, installed Syncthing via Homebrew. Discovered the macOS Homebrew package starts with `--no-browser --no-restart` flags but crucially **picks a random GUI port** in v2 (61696 in this case) instead of the documented default of 8384 — which is actually fine because port 8384 was already in use by an SSH `LocalForward` from a separate tunnel.

Pairing: exchanged device IDs, accepted folder share. Vault content propagated mini PC → Mac in seconds.

### 11. Obsidian Local REST API plugin + reverse SSH tunnel

Plugin runs on the Mac and binds to `127.0.0.1:27123` (HTTP) only — unreachable from the mini PC over Tailscale despite the private-mesh perimeter. Two options surfaced (change plugin bind to `0.0.0.0`, or reverse-tunnel through SSH); chose **reverse SSH tunnel** because it required zero plugin reconfiguration.

First validation used a manual `ssh -N -R 27123:127.0.0.1:27123 kos@...`. Confirmed `HTTP 200` from mini PC against `http://127.0.0.1:27123/`. Then made the tunnel persistent via:

- A dedicated SSH key `~/.ssh/obsidian_tunnel` on the Mac (passphrase-less, just for the tunnel)
- `authorized_keys` entry on the mini PC for that key only
- `autossh` (Homebrew) wrapped in a `launchctl` LaunchAgent at `~/Library/LaunchAgents/com.user.ssh-tunnel-obsidian.plist`

The agent uses `KeepAlive=true` + `ServerAliveInterval=30` so the tunnel survives Mac sleep/wake and network changes.

### 12. mcp-obsidian wrapper for env-aware host/port

The `mcp-obsidian` 0.2.2 PyPI package only reads `OBSIDIAN_API_KEY` from env; it hardcodes `https://127.0.0.1:27124` (HTTPS) as protocol/host/port. Our setup needs HTTP on port 27123 (reverse tunnel doesn't carry HTTPS without cert work).

Solution: wrote a small Python wrapper at `~/.local/bin/mcp-obsidian-wrapped` that monkey-patches `mcp_obsidian.obsidian.Obsidian.__init__` to read `OBSIDIAN_PROTOCOL`, `OBSIDIAN_HOST`, `OBSIDIAN_PORT`, `OBSIDIAN_VERIFY_SSL` from env before delegating to the original constructor. Pipx package install stays untouched, so future `pipx upgrade` doesn't blow away the customization.

### 13. MCP registration in Claude Code

```bash
claude mcp add obsidian \
  -e "OBSIDIAN_API_KEY=..." \
  -e "OBSIDIAN_HOST=127.0.0.1" \
  -e "OBSIDIAN_PORT=27123" \
  -e "OBSIDIAN_PROTOCOL=http" \
  -- /home/kos/.local/bin/mcp-obsidian-wrapped
```

`claude mcp list` immediately reported `obsidian: ... - ✓ Connected`. After a Reload Window in VS Code, the deferred tools `mcp__obsidian__*` (12 of them: list_files, get_file_contents, simple_search, complex_search, append, patch, delete, periodic_note, recent_changes, recent_periodic_notes, list_files_in_dir, batch_get_file_contents) became loadable via ToolSearch.

### 14. API key rotation + endpoint testing

While bootstrapping, the original API key was pasted into chat. After the setup verified end-to-end, rotated the key via Obsidian → Settings → Local REST API → "Reset All Cryptography". Updated both `~/.openclaw/op-env-cached.sh` and the Claude Code MCP registration with the new 64-char key.

A debug rabbit hole: initial `curl http://127.0.0.1:27123/` returned `HTTP 200` regardless of the API key sent (or whether one was sent at all). Discovered the root endpoint `/` is unauthenticated — it's a plugin-info ping. The right authenticated test is `curl ... /vault/`, which correctly returns 401 for missing/wrong keys and 200 for valid ones. Updated mental model: never trust `/` for auth verification on Obsidian Local REST API.

Also discovered that an accidental double-paste of the API key (resulting in a 128-char string in env) was being silently accepted by `/` (no auth required) but rejected by `/vault/`. A simple truncation `${OBSIDIAN_API_KEY:0:64}` recovered the real 64-char key from the duplicated value.

### 15. Slash command + snapshot script for cross-interface use

To make the vault context accessible from non-Claude-Code interfaces (claude.ai web, Anthropic mobile app), shipped two utilities:

- `~/.claude/commands/obs.md` — slash command `/obs` that reads the latest daily, recent decisions, and inbox status and produces a compact summary. Works in any Claude Code session.
- `~/Obsidian-Personal/Templates/snapshot.sh` — bash script that emits a single markdown blob containing recent dailies (last 7 days) + all decisions + all inbox items. Designed to be piped through `pbcopy` on Mac and pasted into a non-Claude-Code chat surface.

The script lives inside the vault so Syncthing replicates it to the Mac automatically.

## Key Discoveries

- **OpenClaw 2026.4.24 expects specific bootstrap filenames** (`BOOTSTRAP.md`, not `BOOT.md`). Custom-named bootstrap files are silently ignored and the agent boots without context.
- **`openclaw.json` rewrite-on-shutdown** behavior: graceful gateway restarts re-serialize the in-memory config, so direct file edits while the gateway is running get clobbered. The CLI commands (`openclaw plugins disable`, `openclaw cron edit`, `openclaw mcp add`) use a safe-merge path that survives. Direct JSON edits should happen with the gateway stopped.
- **`embeddedHarness.runtime` and `model.primary` are independent**. Even if the primary model is Codex, an agent with `embeddedHarness: google-gemini-cli` runs inside a Gemini CLI subprocess and inherits its CLI flag bugs (e.g. the upstream OpenClaw 2026.4.24 `--skip-trust` regression).
- **Obsidian Local REST API root endpoint `/` is unauthenticated**. Always test against `/vault/` or another authenticated endpoint when verifying that a Bearer token actually works. A misleading `HTTP 200` from `/` masked an incorrect key for several iterations during this session.
- **`mcp-obsidian` 0.2.2 only honors `OBSIDIAN_API_KEY` from env**. Host/port/protocol need a wrapper or a monkey-patch. Worth a PR upstream eventually.
- **Reverse SSH tunnel + autossh + launchd** is a clean alternative to changing an app's bind address when you control both endpoints (and have a Tailscale or VPN perimeter). Trade-off: persistent process on the client side (Mac) but zero config on the server side (Obsidian plugin).
- **Specialist agents need real domain SOULs to act as specialists**. The boilerplate "be helpful" SOUL is insufficient — without explicit boundaries (what's NOT yours), specialists drift into generalist territory and Master Control's delegation routing collapses.

## Configuration Changes

- New file: `~/.openclaw/memory/FLEET-DRIVE.md` (Drive folder registry, permission table, output convention)
- Edited: 4 specialist SOUL.md files (frontend-specialist, backend-expert, cso, orchestration-architect)
- Edited: `~/.openclaw/workspace-orchestrator/SOUL.md` (folder IDs, recurring duties)
- Created/renamed: 8 `BOOTSTRAP.md` files across all workspaces
- Edited: `~/.openclaw/openclaw.json` (cross-provider fallback chains, removed orchestrator embeddedHarness, bonjour disabled via correct CLI path)
- Six new cron jobs registered (Daily Briefing, Evening Digest, Backend Schema, CSO Audit, Frontend UX, OrchArch System Design)
- Daily docs elaboration cron: model changed to `openai-codex/gpt-5.4`
- New file: `~/Obsidian-Personal/{CLAUDE.md, README.md, Templates/Daily.md, Templates/snapshot.sh}` + empty `Daily/`, `Decisions/`, `Inbox/`
- New file: `~/job-desk/CLAUDE.md` (cross-project context, vault references)
- New service: `~/.config/systemd/user/syncthing.service` (Syncthing on mini PC)
- Mac LaunchAgent: `~/Library/LaunchAgents/com.user.ssh-tunnel-obsidian.plist` (autossh)
- New SSH key: `~/.ssh/obsidian_tunnel` (Mac), corresponding `authorized_keys` entry on mini PC
- New env vars in `~/.openclaw/op-env-cached.sh`: `OBSIDIAN_API_KEY`, `OBSIDIAN_HOST`, `OBSIDIAN_PORT`
- Pipx-installed package: `mcp-obsidian` 0.2.2
- New file: `~/.local/bin/mcp-obsidian-wrapped` (Python wrapper, env-aware)
- Claude Code MCP registration: `~/.claude.json` mcpServers.obsidian
- New slash command: `~/.claude/commands/obs.md` → invocable as `/obs`

## Final State

- Multi-agent fleet operationally ready: 4 specialists with distinct domain SOULs, MC with explicit folder authority, 6 pilot cron jobs registered, all delivering to Telegram + Drive subfolders.
- Obsidian vault live: `~/Obsidian-Personal/` ↔ Mac (Syncthing P2P) ↔ Cloud (Obsidian Sync).
- Local REST API plugin running on Mac, reverse-tunneled to mini PC via persistent autossh+launchd.
- MCP server registered in Claude Code; tools `mcp__obsidian__*` (12) callable from any new Claude Code session in the workspace.
- API key rotated; the original (briefly shared in chat) is dead.
- `/obs` slash command + `obsidian-snapshot.sh` available for vault context retrieval from any interface.

## Open Questions

- The Mac's Syncthing GUI binds to a random port (`61696`) instead of the documented default `8384`. Investigated: this is a v2 Homebrew package quirk; not actually a problem because the `8384` slot was already used by a `LocalForward` for the mini PC's Syncthing GUI. Worth documenting as a "won't fix" in the troubleshooting reference if other users hit it.
- The `mcp-obsidian` upstream maintainer might accept a PR adding env-var support for `HOST`/`PORT`/`PROTOCOL` so future installs don't need the wrapper. Filed as a follow-up; not blocking.
- The `/` endpoint of Obsidian Local REST API being unauthenticated is documented in the plugin code but easy to miss. Worth a sticky note in the troubleshooting doc to "test auth against `/vault/`, not `/`".
- Pilot cron observation window: first run of MC Daily Work Briefing is tomorrow 06:30. Watch for delegation correctness (does MC actually invoke specialists, or does it cannibalize their work?), Telegram delivery success, and Codex Plus quota burn rate.
