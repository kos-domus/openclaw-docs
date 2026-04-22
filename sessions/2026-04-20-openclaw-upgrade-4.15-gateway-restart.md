---
title: "OpenClaw upgrade 2026.4.8 → 2026.4.15: gateway restart, auth gotchas, ACP activation, per-agent auth gap"
date: "2026-04-20"
author: "kos-domus"
status: "processed"
tags: ["configuration", "cli", "security", "troubleshooting", "multi-agent", "setup", "api", "debugging"]
openclaw_version: "2026.4.15"
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-7[1m]"
---

## Objective

Preventive maintenance on the OpenClaw host: bring the CLI and gateway from `2026.4.8` (installed Apr 8) up to the latest stable `2026.4.15`, verify the fleet stayed healthy across the restart, prune the accumulated `openclaw.json.bak*` backups, and capture any surprises the upgrade surfaced. No active incident — this was hygiene work.

## Context

- Host: AceMagic mini PC, Ubuntu 24.04, gateway under user systemd unit `openclaw-gateway.service`.
- Install path: `/home/kos/.npm-global/bin/openclaw` (npm global, no sudo).
- Fleet at start: 8 agent workspaces — `cos` (default, CoS), `orchestrator`, `cso`, `frontend-specialist`, `backend-expert`, `orchestration-architect`, `family` (Kai), `wip` (Mr Wolf). Plus two orphan dirs on disk (`main`, `katia`) not listed in `agents.list`.
- Model routing at start: `agents.defaults.model.primary = "openrouter/auto"` with fallback chain `gemini-3.1-pro-preview → gemini-2.5-flash → claude-cli/opus-4.6 → openai-codex/gpt-5.4 → openai-codex/gpt-5.4-mini → zai/* → openrouter/*`. Every explicit agent overrode primary to `openai-codex/gpt-5.4` or `gpt-5.4-mini`.
- Constraint: Claude CLI can no longer ride the subscription plan for agent workloads — Anthropic traffic now bills as API, which is prohibitively expensive. Anthropic is therefore a deep-fallback slot only, never primary.
- Seven `openclaw.json.bak*` files had accumulated in `~/.openclaw/` (oldest Apr 3).

## Steps Taken

### 1. Pre-upgrade state audit

Checked running binary, config health, and backup clutter before touching anything.

```bash
openclaw --version                                    # OpenClaw 2026.4.8 (9ece252)
npm view openclaw version                             # 2026.4.15
ls -la ~/.openclaw/openclaw.json*                     # 7 bak files
systemctl --user status openclaw-gateway --no-pager   # active, 6 days uptime, unit Description "(v2026.4.8)"
```

**Result**: running CLI and gateway both on `2026.4.8` (7 patches behind). Gateway PID 1704, uptime 520 632 s (≈6 days). `openclaw.json` parses clean.

### 2. Fetched release notes for 4.9 through 4.15

Pulled every intervening release from GitHub via `gh release view v<ver> --json name,body`.

**Result**: no hard breaking changes. Notable behavior shift in 4.15 — `dreaming.storage.mode` default flipped `inline → separate`, so dreaming phase blocks now land in `memory/dreaming/{phase}/YYYY-MM-DD.md` instead of being injected into daily memory files. Anthropic defaults, `opus` aliases, Claude CLI defaults, and bundled image understanding now resolve to **Claude Opus 4.7**. Relevant fixes for this fleet: cron `NO_REPLY` sentinel leak (4.15), WhatsApp reconnect/creds-race (4.10/4.11/4.15), Telegram forum-topic names persisting across restart (4.14), skills cache invalidation on `skills.*` config writes preventing "Tool X not found" loops (4.15), SIGUSR1 restart-loop fix on plugin auto-enable writes (4.15), multiple SSRF/shell/allowlist hardenings (4.10–4.14).

### 3. Ran the npm global update

```bash
npm install -g openclaw@latest
openclaw --version   # OpenClaw 2026.4.15 (041266a)
```

**Result**: install delta was large — `added 24 packages, removed 158 packages, changed 767 packages`. Matches the 4.12/4.15 work that localized bundled plugin runtime deps to their owning extensions and trimmed the published docs payload. Config still parsed (`node -e 'JSON.parse(...)'` check).

### 4. Ran `openclaw doctor` (read-only)

Captured diagnosis before any `--fix`/`--repair`.

**Result**: no critical issues. Surface findings:

| Finding | Severity | Action |
|---|---|---|
| `anthropic:claude-cli` auth profile missing for agent `main` | low | see step 5 (gotcha — deferred to manual run) |
| 2 orphan agent dirs: `main`, `katia` (no `agents.list` entry) | informational | leave, not touched — may be intentional |
| 2/5 recent `cos` sessions missing transcripts, 3 orphan `.jsonl` files | low | leave, non-blocking |
| Service command does not include the `gateway` subcommand | cosmetic | **do NOT run `doctor --repair`** — would overwrite the custom `start-gateway.sh` + op-env drop-in |
| Service PATH missing volta/asdf/bun/nvm/fnm/pnpm dirs | cosmetic | Kos does not use any of those runtimes |
| Telegram in first-time setup mode (group policy) | informational | pre-existing config choice |

Also confirmed healthy: Telegram `ok (@Kos_OC_bot) (90ms)`, WhatsApp `linked (auth age 7m)`, 8 routable agents, 110 eligible skills, 59 plugins loaded with 0 errors.

### 5. Ran `openclaw models auth login --provider anthropic --method cli --set-default` — auth profile gotcha

The doctor recommendation was run verbatim in Kos's interactive terminal (the command requires a TTY). Output:

```
Updated ~/.openclaw/openclaw.json
Auth profile: anthropic:claude-cli (claude-cli/oauth)
Default model set to claude-cli/claude-opus-4-7
◇ Provider notes
│ Claude CLI auth detected; switched Anthropic model selection to the local Claude CLI
│ backend.
│ Existing Anthropic auth profiles are kept for rollback.
```

**Result**: the `--set-default` flag did more than expected — it rewrote `agents.defaults.model.primary` from `openrouter/auto` to `claude-cli/claude-opus-4-7` (the exact expensive tier we wanted to avoid), while also adding six new entries under `agents.defaults.models` for the full claude-cli catalog (`claude-opus-4-7`, `claude-sonnet-4-6`, `claude-opus-4-6`, `claude-opus-4-5`, `claude-sonnet-4-5`, `claude-haiku-4-5`, all without aliases). The auth profile itself was correctly added. All explicit agent-level `primary` fields were untouched, so only agents inheriting defaults (e.g. the orphan `main` dir) would have been exposed.

A fresh `openclaw.json.bak` was auto-created at the same timestamp, giving a clean rollback target.

### 6. Surgical revert of the default primary

Instead of a full `.bak` restore (which would have removed the intentional auth profile addition), reverted just the one line via `Edit`:

```diff
   "agents": {
     "defaults": {
       "model": {
-        "primary": "claude-cli/claude-opus-4-7",
+        "primary": "openrouter/auto",
         "fallbacks": [ ... ]
```

Validated with `node -e 'JSON.parse(...)'` → `primary: openrouter/auto` and `parse OK`. The six new dictionary entries under `agents.defaults.models` were left alone — they have no alias and don't route traffic unless explicitly referenced elsewhere.

**Result**: default routing restored without throwing away the auth profile.

### 7. Restarted the gateway

```bash
systemctl --user restart openclaw-gateway
```

**Result**: clean restart. New PID 1146359. Gateway ready in 17.4 s. Log line:

```
[gateway] ready (7 plugins: acpx, browser, device-pair, phone-control, talk-voice, telegram, whatsapp; 17.4s)
[telegram] [work] starting provider (@Workspace00_bot)
[telegram] [default] starting provider (@Kos_OC_bot)
[whatsapp] [default] starting provider (+393249847157)
[whatsapp] Listening for personal WhatsApp inbound messages.
[plugins] embedded acpx runtime backend ready
```

Running binary confirmed as `openclaw@2026.4.15` via `npm ls -g openclaw`. `/proc/<pid>/exe` symlinks to `/usr/bin/node`, loading the updated module path.

### 8. ACP regression check

Probed the long-documented `openclaw acp client` banner-pollution bug (tracked in `docs/troubleshooting/common-errors.md`: "`Unexpected token '│'` — ACP startup banner pollutes the JSON stream").

```bash
(echo '{"jsonrpc":"2.0","method":"initialize","params":{...},"id":1}'; sleep 4) \
  | timeout 10 openclaw acp client --verbose 1>/tmp/acp-stdout 2>/tmp/acp-stderr
```

**Result**: on 4.15, `stdout` produced **0 bytes** during an `acp client` run — banner no longer appears on stdout, and `[acp-client]` diagnostic lines route correctly to stderr. The `--help` subcommand still emits the banner on stdout (as normal). The documented symptom (`Unexpected token '│'`) should not recur under 4.15.

### 9. Pruned old config backups

Per Kos: keep only the latest (`.bak` @ 11:37 today) plus the named Apr-7 snapshot (`.bak-2026.4.2`) as a historical pinpoint.

```bash
cd ~/.openclaw && rm -v \
  openclaw.json.bak.1 openclaw.json.bak.2 openclaw.json.bak.3 \
  openclaw.json.bak.4 openclaw.json.bak-2026.4.1
```

**Result**: 5 removed, 2 kept.

```
-rw------- 1 kos kos 19382 Apr 20 11:41 openclaw.json            # live, reverted primary
-rw------- 1 kos kos 19025 Apr 20 11:37 openclaw.json.bak        # pre-auth-login rollback
-rw------- 1 kos kos 12451 Apr  7 06:39 openclaw.json.bak-2026.4.2
```

## Configuration Changes

Net diff against pre-session state (`openclaw.json.bak` @ 11:37):

- **Added** under `auth.profiles`: `anthropic:claude-cli` (oauth, from Claude CLI local auth). Kept — desirable so the `claude-cli/opus-4.6` fallback slot can authenticate when Gemini tiers are exhausted.
- **Added** under `agents.defaults.models`: six entries — `claude-cli/claude-opus-4-7`, `claude-cli/claude-sonnet-4-6`, `claude-cli/claude-opus-4-6`, `claude-cli/claude-opus-4-5`, `claude-cli/claude-sonnet-4-5`, `claude-cli/claude-haiku-4-5` — no aliases, no routing change. Kept as harmless catalog.
- **Reverted** `agents.defaults.model.primary`: `"claude-cli/claude-opus-4-7"` → `"openrouter/auto"` (back to pre-session value).

No changes to explicit agent `primary` fields (cos, orchestrator, cso, frontend-specialist, backend-expert, orchestration-architect, family, wip).

## Key Discoveries

1. **The 4.8 gateway had been rejecting 21 plugins for weeks.** Journalctl from just before the restart showed the `gateway/reload` subsystem warning that `bluebubbles, discord, feishu, googlechat, irc, line, matrix, mattermost, memory-lancedb, msteams, nextcloud-talk, nostr, qqbot, synology-chat, tlon, twitch, voice-call, whatsapp, zalo, zalouser` all required `>=2026.4.10` and were being skipped on every config reload. `whatsapp` being in that list is significant — the live WhatsApp listener was staying up via the already-established runtime from the original 4.8 boot, but any config reload would not have re-registered it. The ≥4.10 requirement has been in place since the 4.10 release on 2026-04-11, so the host had been in a degraded reload state for roughly 9 days. This alone justified the update.

2. **`--set-default` on `openclaw models auth login` is overloaded.** The flag name reads like "set this as the default auth profile for this provider", but it actually also rewrites `agents.defaults.model.primary` to point at the most recent model from the provider (in 4.15 this is Claude Opus 4.7, per the 4.15 default-alias change). Worth flagging to anyone running the command on a fleet where Anthropic should stay in a deep-fallback slot. The safe incantation when only the profile is wanted is plain `openclaw models auth login --provider anthropic --method cli` (no `--set-default`).

3. **4.15 Anthropic default alias change changes the blast radius of mistakes.** The 4.15 changelog note "default Anthropic selections, `opus` aliases, Claude CLI defaults, and bundled image understanding to Claude Opus 4.7" means any implicit `opus` reference anywhere in the config (agent, skill, plugin) now resolves to the top-tier model. On installs that route Anthropic via API, this is a cost-spike waiting to happen. Worth auditing the config for bare `opus` aliases.

4. **ACP banner pollution bug is fixed in 4.15.** The `Unexpected token '│'` symptom recorded in `docs/troubleshooting/common-errors.md` line 80 should be resolved; stdout of `openclaw acp client` is now clean (0 bytes before any real response), with diagnostic lines on stderr. The troubleshooting entry can be updated to "fixed in 2026.4.15".

5. **`doctor --repair` is not safe on this install.** Doctor itself surfaced "Custom or unexpected service edits detected. Rerun with --force to overwrite." The custom `start-gateway.sh` wraps the op-env secret loading (`source op-env-cached.sh`) plus the service drop-ins (`env.conf`, `op-token.conf`). A naive `--repair --force` would overwrite the unit command back to the stock `openclaw gateway` path and break secret injection. Document this as a known landmine.

6. **Restart was the actual activation step, not the npm install.** The gateway process kept running the cached 4.8 runtime for 6 days while 4.15 sat on disk after `npm install -g`. Anyone who "updates OpenClaw" without restarting the gateway unit is still on the old runtime. Worth surfacing in the cron/automation reference — it may be the right moment to add a systemd `ExecStartPost` or a post-update hook.

7. **ACP banner pollution is fully resolved under real clients.** The `formulahendry.acp-client` VS Code extension now connects against `npx openclaw acp` without the `Unexpected token '│'` error that blocked the April 1 attempt. Default `acp.agents` entry ships with `"OpenClaw": {"command": "npx", "args": ["openclaw", "acp"]}` — zero custom config needed beyond installing the extension in the SSH-Remote host where the gateway runs. Worth adding to `guides/` as a one-page ACP/VS Code setup.

8. **`openclaw models auth login` is per-gateway, not per-agent.** The CLI has `--agent <id>` on `openclaw models auth` (parent) but only for `order get/set/clear` subcommands, not for `login`. A fresh OAuth login therefore writes new credentials to only the *active* agent workspace + the global `openclaw.json` auth declaration. Any other agent whose per-agent `auth-profiles.json` has the same provider already will keep using its stale tokens silently. This is the root cause of the multi-agent `refresh_token_reused` cascade. Candidate upstream issue: `openclaw models auth login --provider <p> --agent <id>` should target a single agent's auth-profiles.json; a bulk mode `--all-agents` could propagate a fresh profile to every workspace in one shot. Until fixed, multi-agent installs are forced to share one refresh token or manually edit per-agent auth files.

9. **Declaration ≠ credentials in OpenClaw auth.** The top-level `openclaw.json` `auth.profiles.<name>` entries are just declarations: `{provider, mode}` stubs. Actual access/refresh tokens live inside each agent's `~/.openclaw/agents/<id>/agent/auth-profiles.json`. Removing a stale per-agent profile does **not** make the agent fall through to the global declaration — it just leaves the agent without credentials for that named profile. Agents don't silently try alternative profile names for the same provider. This trips up anyone trying to "just delete the stale profile" to force a fresh one; the actual remediation needs token material copied, not removed.

10. **OAuth refresh tokens are single-use; 4 agents sharing one is a race.** Codex OAuth issues a new refresh token on every refresh call. If N agents share the same credentials and any two trigger refresh near-simultaneously, the first wins, the others get `refresh_token_reused`. The gateway's lane parallelism (`lane=main`, `lane=session:agent:wip:main`, multiple subagent lanes) virtually guarantees overlap. This is why the cascade appears every few minutes in the logs even when no one is actively using Codex — background heartbeats, subagent announces, and cron-triggered runs all probe Codex and race the refresh. The only robust fix is independent OAuth sessions per agent, which ties back to the CLI gap above.

11. **ACP-driven failures are louder than model-fallback failures.** On Telegram or WhatsApp, the auth cascade → Gemini fallback chain is invisible to the user — they just get a reply ~30 s later. On ACP, the client is staring at a `agent.wait` RPC and actively surfaces intermediate errors (`All models failed (3): ...`) as user-visible cards. When fallback recovers via a second retry or different lane, the client may not re-synthesize a clean state; hence the "Internal error" card even though logs show `candidate_succeeded` soon after. ACP is the most brittle surface for auth misconfiguration and should be treated as the canary for it, not the last surface to notice.

12. **"Model choice ≠ plan ownership" is a useful architectural split.** Kos's OpenAI Pro plan is worth ~$200/mo of Codex capacity dedicated to cos's workflow. Every other agent is free to route through Gemini (which is paid by usage but effectively free for his volume) without any tradeoff. Changing orchestrator's primary to Gemini cost nothing semantically and unblocked ACP completely. Worth promoting as a general pattern in `concepts/agent-fleet-design.md`: **match each agent's primary model to the identity of the paying account, not to a "best model" default**. The per-agent model fields in `openclaw.json` were designed exactly for this — they had been underused.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `openclaw models auth login … --set-default` silently promoted Claude Opus 4.7 to default primary model | `--set-default` rewrites both the auth profile **and** `agents.defaults.model.primary` using the provider's newest bundled model (Opus 4.7 in 4.15) | Surgical `Edit` to revert `agents.defaults.model.primary` back to `openrouter/auto`. JSON re-validated with `node -e 'JSON.parse(...)'` before restart. Full `.bak` also existed as fallback. |
| `openclaw models auth login` fails non-interactively: `models auth login requires an interactive TTY` | Command expects a TTY for the OAuth browser flow prompt | Run in a terminal, not from a subprocess. No workaround needed, just a scheduling note. |
| Gateway log (pre-restart): `config reload skipped (invalid config): plugins.entries.whatsapp: plugin requires OpenClaw >=2026.4.10, but this host is 2026.4.8` (and 20 other plugins) | Running CLI was 2026.4.8; plugin manifests bundled with the installed npm package required >=2026.4.10 after the 4.10 refactor | Upgrade resolved it — after restart, 7 of those plugins loaded cleanly (acpx, browser, device-pair, phone-control, talk-voice, telegram, whatsapp). The remaining ones are channels Kos doesn't use. |
| `[openai-codex] Token refresh failed: 401 refresh_token_reused` on every 2–3 minutes for `lane=session:agent:wip:*`, `lane=main`, subagent lanes | 4 agents (cos, wip, family, orchestrator) share the same Codex OAuth refresh token in their per-agent `auth-profiles.json` files; refresh tokens are single-use, so concurrent refresh attempts race and all-but-one fail | Partial: re-auth via `openclaw models auth login --provider openai-codex --method cli` added a fresh profile to cos only. Other 3 agents still on stale token. Parked pending a `--agent` flag on `login` or manual per-agent token propagation (risks same race). **Gemini 3.1 Pro fallback catches 100% of failures**, so user-visible impact is ~30–60 s added latency, not dropped messages. |
| VS Code ACP: messages surface as "internal error" before Gemini fallback completes | ACP client times out or shows the 401 before the gateway's fallback chain finishes (typically ~30–60 s) | Inherent to the Codex auth cascade above; will self-resolve once per-agent auth is straightened. Workaround: wait for the Gemini reply to arrive anyway, or switch the affected agents' `model.primary` to `google-gemini-cli/gemini-3.1-pro-preview` in `openclaw.json` to skip the failing Codex attempt entirely. |
| `openclaw models auth login --provider openai-codex --method cli --agent wip` → `error: unknown option '--agent'` | `--agent` only lives on `openclaw models auth` parent and only applies to `order` subcommands, not `login` | No CLI workaround on 2026.4.15; escalate upstream as a multi-agent setup gap. |

## Final State

- **CLI**: `OpenClaw 2026.4.15 (041266a)` at `/home/kos/.npm-global/bin/openclaw`.
- **Gateway**: systemd unit `openclaw-gateway.service` active, PID 1146359, 4.15 runtime, 7 plugins loaded.
- **Channels live**: Telegram (`@Kos_OC_bot` default + `@Workspace00_bot` work), WhatsApp (+393249847157), ACP embedded backend ready.
- **Fleet**: 8 agents routable (cos, orchestrator, cso, frontend-specialist, backend-expert, orchestration-architect, family, wip).
- **Config**: `agents.defaults.model.primary = "openrouter/auto"` (restored). Anthropic `claude-cli` auth profile present for deep-fallback slot. Explicit agent primaries untouched.
- **Backups**: `openclaw.json.bak` (11:37 today) + `openclaw.json.bak-2026.4.2` (Apr 7) only.
- **Memory**: new feedback entry `feedback_anthropic_api_cost.md` added so future sessions don't repeat the `--set-default` mistake or promote Anthropic to primary.
- **ACP**: confirmed working end-to-end via `formulahendry.acp-client` VS Code extension → `openclaw acp --session agent:orchestrator:main` → Master Control on Gemini 3.1 Pro. VS Code is now a usable surface against the local gateway.
- **Agent model routing**: orchestrator (Master Control) now primary on Gemini 3.1 Pro; cos preserves Codex for the OpenAI Pro plan. Other workers (cso/frontend/backend/orch-arch) remain on their prior `openai-codex/gpt-5.4-mini` — will continue to hit the refresh race until the underlying CLI gap is resolved, but they're not on any user-facing latency path.

### 10. Activated the ACP VS Code extension against the 4.15 gateway

With the banner-pollution bug fixed, attempted to actually use the `formulahendry.acp-client` VS Code extension (installed on the mini PC, reached via VS Code Remote-SSH from Kos's Mac). The extension ships with a default `OpenClaw` entry in `acp.agents`:

```json
"OpenClaw": { "command": "npx", "args": ["openclaw", "acp"], "env": {} }
```

Flow: `Cmd+Shift+P` → `ACP: Connect to Agent` → pick `OpenClaw`. The ACP server spawned cleanly (`pstree` showed `npm exec openclaw acp → sh -c openclaw acp → openclaw-acp`); gateway logs recorded `embedded acpx runtime backend registered (cwd: /home/kos/.openclaw/workspace-cos)` followed by `embedded acpx runtime backend ready`. No `Unexpected token '│'` — the 4.15 fix for the documented ACP banner bug (`docs/troubleshooting/common-errors.md:80`) is confirmed to hold under a real VS Code client, not just a synthetic probe.

**Result**: the ACP session establishes on `workspace-cos`, gateway RPC calls succeed (e.g. `⇄ res ✓ sessions.list 136ms`). ACP banner bug is fully resolved in real-world use. VS Code is now a usable client surface against the local gateway. `troubleshooting/common-errors.md:80` can be marked "fixed in 2026.4.15".

### 11. Codex OAuth `refresh_token_reused` cascade — diagnosis

First ACP message from VS Code did not get through visibly ("internal error"). Gateway logs revealed a different failure mode than the ACP banner bug:

```
[openai-codex] Token refresh failed: 401
  "Your refresh token has already been used to generate a new access token.
   Please try signing in again."
  code: refresh_token_reused
```

followed by cascading `lane task error` entries on `lane=main`, `lane=session:agent:wip:main`, `lane=session:agent:wip:subagent:…`, then eventual `model_fallback_decision: candidate_succeeded` via `google-gemini-cli/gemini-3.1-pro-preview` — delayed by ~30–60 s.

Re-authenticated Codex from a TTY without `--set-default` (memory-feedback respected):

```bash
openclaw models auth login --provider openai-codex --method cli
```

Browser OAuth completed, new profile `openai-codex:alessandrobenedetti90@gmail.com (openai-codex/oauth)` written. Gateway auto-restarted after draining in-flight operations. Yet the `refresh_token_reused` errors continued firing on `wip` lanes after the restart.

**Audit of per-agent auth-profiles.json**:

```
cos           codex profiles = [openai-codex:default (stale), openai-codex:alessandrobenedetti90@gmail.com (fresh)]
wip           codex profiles = [openai-codex:default (stale from March)]
family        codex profiles = [openai-codex:default (stale)]
orchestrator  codex profiles = [openai-codex:default (stale)]
```

Three of the four agents have only the stale profile. Worse, the three stale tokens partially overlap (`rt_CDRWRGbk…` shared between family and orchestrator; wip has its own older `rt_py5DmZj…` last seen in the archived `main` agent from 2026-03).

**Attempted remediation — failed**: Removed `openai-codex:default` from wip/family/orchestrator's auth-profiles.json (with per-file `.bak-2026-04-20-codex-cleanup` backups), hoping agents would fall through to the global `openai-codex:alessandrobenedetti90@gmail.com` profile whose stub exists in the top-level `openclaw.json` `auth.profiles` declaration. Next message produced a new error shape: `No credentials found for profile "openai-codex:default"`. Gemini fallback still caught everything, but the regression meant a **hard** missing-creds failure instead of a retry-able 401. Restored from backups.

**Result (parked)**: Codex OAuth remains broken for wip, family, orchestrator. Gemini 3.1 Pro fallback absorbs 100% of failures; ACP/VS Code shows "internal error" briefly before the fallback completes. cos is fine.

### 12. ACP routing split: Master Control gets Gemini, cos keeps Codex

User-visible testing of ACP via VS Code revealed a worse failure mode than "slow fallback": messages alternated between red "Internal error" cards (when gateway returned `errorCode=UNAVAILABLE: FallbackSummaryError: All models failed (3)`) and indefinite "Stop" spinners (when the agent got stuck in a runaway Gemini loop — logs showed 20+ `cli exec: provider=google-gemini-cli` calls every 3–5 s with prompts up to 2236 chars, no `final_answer` emitted, `agent.wait` calls timing out at the 900 000 ms ceiling). The ACP client was getting intermediate error states before fallback could complete, and some runs were never completing at all.

Architectural decision rather than another token-level remediation: **split ACP routing from cos's model config**. Kos holds an OpenAI Pro plan ($200/mo) generously provisioned for cos's Codex usage, so losing Codex on cos is not acceptable. But no such constraint applies to orchestrator (Master Control), which is only pinned to Telegram work-group routing and has no cost/ownership overlap with cos.

**Config change** — orchestrator (id `orchestrator`, display name Master Control) in `~/.openclaw/openclaw.json`:

```diff
   "id": "orchestrator",
   "model": {
-    "primary": "openai-codex/gpt-5.4",
-    "fallbacks": [
-      "google-gemini-cli/gemini-3.1-pro-preview",
-      "google-gemini-cli/gemini-2.5-flash"
-    ]
+    "primary": "google-gemini-cli/gemini-3.1-pro-preview",
+    "fallbacks": [
+      "google-gemini-cli/gemini-2.5-flash",
+      "openai-codex/gpt-5.4"
+    ]
   },
```

Codex kept as deep fallback on orchestrator since it already has a (stale) profile in its workspace — reach probability is low enough that the race rarely fires, and if Gemini ever goes down completely, Codex is still there as a last resort.

**VS Code extension change** — point the `formulahendry.acp-client` at orchestrator explicitly. Added to User or Workspace `settings.json`:

```json
"acp.agents": {
  "Master Control": {
    "command": "openclaw",
    "args": ["acp", "--session", "agent:orchestrator:main"],
    "env": {}
  }
}
```

Notes on the entry: use `openclaw` not `npx openclaw` (resolves via global install without npm overhead), and the `--session agent:orchestrator:main` flag forces the ACP bridge to route to orchestrator's workspace instead of defaulting to cos.

**Result**: gateway auto-restarted at 15:47:42, ACP backend re-registered, first Master Control message through ACP at 15:49:53 ran `gemini-3.1-pro-preview` cleanly, returned via `candidate_succeeded` at 15:53:25. **No Codex cascade, no `refresh_token_reused` errors, no runaway loop**. ACP/VS Code now produces a normal reply in ~10–15 s. cos remains on Codex for its own Telegram DM and Drive workflows (OpenAI Pro plan preserved).

## Open Questions

- **Orphan agent dirs `main` and `katia`**: are these intentional (paused/stub agents — e.g. a planned Katia bot for the mother) or drift from the 8-agent migration? Kos to decide whether to restore their `agents.list` entries or archive the dirs. Until decided, they stay untouched on disk (sessions/auth preserved but no routing).
- **Anthropic CLI auth login still pending a manual retry.** The `openclaw models auth login --provider anthropic --method cli` command needs a TTY; it was attempted from a non-interactive subprocess and rejected. Kos to run it from a real terminal without `--set-default` to register the auth profile for the orphan `main` agent without touching default primaries. Alternative: decide the `main` agent is truly dead and remove it.
- **Service unit `Description=` text is stale** (`OpenClaw Gateway (v2026.4.8)`). Cosmetic, but `doctor --repair --force` is the only clean way to refresh it, and that would overwrite the op-env-integrated `start-gateway.sh`. Either manually edit the `Description=` line in the unit file, or accept the cosmetic drift. Not urgent.
- **Should a `claw.json` convention emerge for post-update gateway restarts?** Today's finding — that 9 days of 21 plugins silently failing to reload on 4.8 — suggests a `claw doctor --post-update` hook or a cron check for "gateway runtime version ≠ installed CLI version" could catch this class of drift automatically.
- **Audit for bare `opus` aliases.** The 4.15 alias-to-Opus-4.7 change means any skill, plugin, or prompt that references `opus` without a version suffix now routes to the priciest tier. Worth a one-shot grep across `skills/**`, plugins, and session-store prompts.
- **Per-agent Codex OAuth propagation** — the blocker from this session. Three agents (wip, family, orchestrator) remain on dead refresh tokens and rely on Gemini fallback on every call. Kos to decide between: (a) file an upstream issue for `openclaw models auth login --agent <id>`, (b) manually copy cos's fresh profile into the other 3 workspaces and accept the concurrent-refresh race (adds back `refresh_token_reused` noise), or (c) change `model.primary` on those three agents to `google-gemini-cli/gemini-3.1-pro-preview` in `openclaw.json` so they stop attempting Codex at all. Option (c) is the cheapest durable fix if Gemini capability is acceptable for those agents' roles.
