---
title: "Common Errors and Solutions"
slug: "common-errors"
category: "troubleshooting"
tags: ["errors", "troubleshooting", "debugging"]
sources: ["sessions/2026-03-12-ubuntu-usb-setup-for-openclaw.md", "sessions/2026-03-13-acemagic-ubuntu-openclaw-install.md", "sessions/2026-03-14-anthropic-auth-apikey-vs-setuptoken.md", "sessions/2026-03-19-google-workspace-cli-gws-integration.md", "sessions/2026-03-23-handson-8agent-setup.md", "sessions/2026-03-28-1password-full-integration-2.md", "sessions/2026-03-30-kos-bootstrap-cron-jobs-1password-gemini.md", "sessions/2026-03-31-openclaw-update-oauth-models-monitor.md", "sessions/2026-03-31-subscriptions-expense-tracking-3.md", "sessions/2026-03-30-kai-new-capabilities-reminders-audio-2.md", "sessions/2026-03-30-kai-reminders-audio-implementation-3.md", "sessions/2026-03-30-kai-fixes-kos-openclaw-monitor-4.md", "sessions/2026-03-31-kai-cron-briefings-waste-calendar-memory-2.md", "sessions/2026-04-01-openclaw-v31-acp-kos-pipeline-kai-mensa.md", "sessions/2026-04-20-openclaw-upgrade-4.15-gateway-restart.md", "sessions/2026-04-23-spesify-major-rebuild-fixes.md", "memory/reports/openclaw-monitor/2026-04-26.md", "sessions/2026-04-26-openclaw-4.24-upgrade-bonjour-and-fleet-fallback.md"]
last_updated: "2026-04-26"
version: 8
---

# Common Errors and Solutions

Aggregated error catalog from real-world OpenClaw deployments.

## Installation and Setup

| Error | Cause | Solution |
|-------|-------|---------|
| Boot menu doesn't appear | Wrong key for your hardware | Try F2, F7, F10, F11, F12, Del, Esc |
| Windows Autopilot blocks Ubuntu boot | AceMagic auto-provisioning | Press `Shift+F10` → `wpeutil shutdown` |
| `curl: command not found` | Not preinstalled on Ubuntu minimal | `sudo apt install curl` |
| No WiFi detected on AceMagic | MediaTek MT7902 has no Linux driver (until kernel 7.1) | Use Ethernet or USB WiFi dongle |

## Authentication

| Error | Cause | Solution |
|-------|-------|---------|
| `API rate limit reached` | API key rate limits are tier-based, not balance-based | Switch to setup token (`sk-ant-oat01-...`) |
| Bot unreachable after rate limit | All API calls fail in cascade | Edit `~/.openclaw/openclaw.json` directly |
| Onboarding asks for `sk-ant-oat01` but you have `sk-ant-api03` | Wrong auth method selected | Re-run `openclaw onboard`, select correct type |

## Google Workspace / GWS

| Error | Cause | Solution |
|-------|-------|---------|
| OAuth login fails with too many scopes | Testing-mode apps limited to ~25 scopes | Use `gws auth login -s drive,gmail,calendar,sheets` |
| "Google hasn't verified this app" | App in testing mode | Click Advanced → Go to (app name) (unsafe) |
| GWS `--upload` fails with path error | `--upload` requires relative paths from cwd | `cd` into the project directory first |
| Skills not detected by OpenClaw | Missing symlinks or `gws-shared` | Verify symlinks with `ls -la ~/.openclaw/skills/`, ensure `gws-shared` is linked, restart gateway |

## Channel Configuration

| Error | Cause | Solution |
|-------|-------|---------|
| WhatsApp QR code expired | QR codes last 1–2 minutes | Re-run `openclaw channels login` with phone ready |
| Wrong agent responds on WhatsApp | Group JID used as `accountId` instead of `peer` | Use `peer: { "kind": "group", "id": "JID" }` |
| Telegram bot doesn't respond in group | Privacy mode enabled in BotFather | `/setprivacy → Disable` |
| "not-allowed" in Telegram logs | Group not in allowlist | Add group ID to `allowGroups` in channel config |
| "Skipped bindings already claimed" | Broad binding from another agent matches first | Make bindings more specific or reorder |
| Can't find Telegram topic thread IDs | Gateway consumes polling updates | Stop gateway → send messages → query Bot API → restart |
| `threadId` in binding causes "Invalid input" | OpenClaw schema doesn't support `threadId` in peer | Bind the entire group to one orchestrator agent |

## 1Password CLI

| Error | Cause | Solution |
|-------|-------|---------|
| Fallback API keys show up in normal runs | OAuth/CLI subscription provider failed or was unavailable | Keep the keys in 1Password, but fix the primary provider path and re-run the job after verifying the subscription auth |


| Error | Cause | Solution |
|-------|-------|---------|
| `op whoami` → "not signed in" | Session tokens are process-bound | Use service account instead of manual signin |
| `~/.1password/agent.sock` not created | Desktop app snap/apt conflict | Use service account (skip desktop app integration) |
| `op item create --field` → "unknown flag" | op v2 uses assignment syntax | Use `'field=value'` instead of `--field` |
| `op read` with parentheses in name → "invalid character" | Special characters in item names break `op://` | Use item UUIDs instead of display names |
| `op item get` → "vault query must be provided" | Service accounts need explicit vault | Add `--vault VaultName` or use `op://Vault/...` syntax |
| Desktop app crashes on SSH | No display (`$DISPLAY` not set) | Use service account for headless access |

## Agent Configuration

| Error | Cause | Solution |
|-------|-------|---------|
| Custom `.md` files overwritten | `openclaw agents add` creates default templates | Write custom files **after** running the command |
| Heredoc won't close | Backticks in content confuse bash parser | Use alternative delimiter: `<< 'SKILLEOF'` |
| File content truncated during paste | Terminal buffer limit for multi-line paste | Use `python3 -c "open('path','w').write('''content''')"` |
| Unicode curly quotes cause errors | Claude UI generates typographic quotes | Replace with ASCII: `'` and `"` |
| LLM suggests nonexistent config params | Model hallucination | Always verify against official documentation |
| `autoApprove` rejected as invalid config | The key is not part of the OpenClaw schema | Use `openclaw approvals allowlist ...` instead of inventing a JSON flag |
| Cron job `exec` times out waiting for approval | The job runs in chat without an interactive approval path | Add an explicit allowlist entry for the agent and restart the gateway if needed |
| `openclaw acp client` fails with `Unexpected token '│'` | Old ACP startup banner polluted the JSON stream on stdout | Fixed in `2026.4.15`. Upgrade OpenClaw, restart the gateway, and re-test before debugging the client |
| `openclaw models auth login --set-default` silently changes the fleet default model | The flag updates the auth profile **and** rewrites `agents.defaults.model.primary` to the provider's latest bundled model | Use `openclaw models auth login --provider <provider> --method <method>` without `--set-default` unless you explicitly want to change fleet-wide default routing |
| `plugin requires OpenClaw >=...` appears during config reload | The npm package on disk is newer than the running gateway process, or the host is genuinely behind plugin minimum version requirements | Upgrade if needed, then restart the gateway. `npm install -g openclaw@latest` alone does not switch the live runtime |
| `openclaw doctor --repair` wants to rewrite a custom gateway service | Doctor detected manual edits such as wrapper scripts, env sourcing, or service drop-ins | Do **not** force repair blindly. Review the unit first, or you can wipe the custom startup path that injects secrets or local env |

## Channels and Providers

### Diagnostic principle: check the gateway before the channel

When a channel symptom recurs on a regular interval — WhatsApp 499s every ~30 min, Telegram silent gaps every ~5 min, etc. — **check gateway uptime first**. A regular cadence is almost never a provider-side timeout. It's almost always:

- systemd restart backoff (`Restart=on-failure` defaults: 100ms → 1s → 10s → 30s → 1min → 5min → 30min)
- a watchdog plugin re-initialising on a fixed schedule
- a credential refresh hitting the same expiry every cycle

Recipe before touching any channel credential:

```bash
# 1. Has the gateway been restarting?
systemctl --user show openclaw-gateway.service \
  -p NRestarts -p ActiveEnterTimestamp --no-pager

# If NRestarts is non-zero or ActiveEnterTimestamp is recent, the gateway
# itself is the suspect — not the channel. Pull the logs:
journalctl --user -u openclaw-gateway.service --since "1 hour ago" \
  | grep -E "Unhandled|exited|FAILURE|Scheduled restart"

# 2. Was a stability bundle written?
ls -lt ~/.openclaw/logs/stability/ | head -5

# 3. Only after ruling out gateway-level crashes, inspect the channel:
journalctl --user -u openclaw-gateway.service --since "1 hour ago" \
  | grep -i <channel-name>
```

Real example from 2026-04-26: the WhatsApp 499 reconnect loop reported by Kos was downstream of a bonjour plugin crash that triggered `Scheduled restart job, restart counter is at N`. Each gateway restart re-initialised the WhatsApp provider mid-handshake; the resulting reconnect attempts surfaced as 499s at the channel layer. Disabling the failing plugin fixed the WhatsApp symptom without re-pairing the Baileys session.

> ⚠️ **Don't delete a Baileys session, rotate a Telegram token, or re-login OAuth on a channel that's recurring at a fixed cadence until you've ruled out a gateway-side restart loop.** You'll spend the credential rotation budget on a symptom and the underlying cause will reproduce within minutes.

### WhatsApp reconnect loop (status 499)

| Symptom | Cause | Recovery |
|---|---|---|
| WhatsApp channel logs `connection closed status=499` and reconnects every ~30 minutes; inbound message count stays at zero | Baileys session secret has been invalidated server-side (typically: another device logged in with the same number, or the QR pairing was revoked). The gateway keeps retrying the same dead session | Stop the WhatsApp channel, delete the cached Baileys session (`~/.openclaw/wa-session-<agent>/` or the path defined in `openclaw.json` channel config), restart the channel, scan a fresh QR from the linked-devices screen on the phone |

Diagnostic checks before deleting the session:

```bash
# 1. Confirm the loop is auth, not network
openclaw logs --channel whatsapp --since 1h | grep -E '499|qr|paired|disconnected'

# 2. Check the session file timestamp — if it's days old and never refreshed
#    after the last 499, the secret is dead
ls -la ~/.openclaw/wa-session-*/creds.json

# 3. Verify the device is still paired on the phone (Settings → Linked devices)
```

> ⚠️ **Don't simply restart the gateway.** A dead Baileys session reproduces the loop instantly. The fix is re-pairing, not a process bounce.

### OpenAI Codex auth failures

| Error string | Cause | Recovery |
|---|---|---|
| `refresh_token_reused` | The same Codex refresh token was used by two agent processes simultaneously, and OpenAI invalidated the family on the second use | `openclaw auth refresh codex` from a single process; if it still fails, delete the cached token (`~/.config/openclaw/codex-*.json`) and run `openclaw models auth login --provider openai-codex` |
| `credential unavailable` | The auth profile in `openclaw.json` references credentials that were never propagated to that worker's workspace | Re-run the login per-agent: `OPENCLAW_AGENT=<agent> openclaw models auth login --provider openai-codex` |

Prevention:

- **One refresh token per worker.** If multiple agents need Codex, give each its own auth profile rather than sharing one.
- **Pair Codex-primary jobs with a non-Codex fallback.** A single Codex auth blip otherwise fails the whole job. See [Agent Fleet Reference — Cross-provider fallback rule](../reference/agent-fleet-reference.md#cross-provider-fallback-rule-apr-25-2026).
- **Daily auth health check** in Kos or equivalent ops agent — a `refresh_token_reused` once is recoverable, twice in a day is the signal to redesign the sharing.

### Gemini saturation: "All models failed"

| Symptom | Cause | Recovery |
|---|---|---|
| Job exits with `All models failed`, every model in the chain returned 5xx or rate-limit | The fallback chain is all Gemini variants and the provider is regionally saturated | Add a non-Gemini fallback (Codex-mini or Anthropic Sonnet 4.6) at the bottom of the chain. Re-run the job after ~30 min if the saturation is transient |

> ⚠️ **A two-Gemini chain is not a fallback chain.** Every entry on the cross-provider fallback table in the [Agent Fleet Reference](../reference/agent-fleet-reference.md#cross-provider-fallback-rule-apr-25-2026) is for this exact scenario. Apply that template to any job whose failure has a real cost.

## Security

| Error | Cause | Solution |
|-------|-------|---------|
| `~/.claude` is world-readable (775) | Default installation behavior | `chmod 700 ~/.claude` |
| IP restriction doesn't work in Google Cloud | Using LAN IP instead of public IP | Use `curl ifconfig.me` to find public IP |
| OAuth flow won't complete on headless server | No browser available | Use SSH tunnel: `ssh -L 8080:localhost:8080 user@server` |

## General Debugging

### Recover an overwritten file from Claude Code transcripts

If a file was accidentally overwritten during a Claude Code assisted session, the local JSONL transcript can act as a last-resort recovery log.

| Error | Cause | Solution |
|-------|-------|---------|
| A working file was overwritten and only generated output survived | The real source of truth lived in a build or `dist/` directory, and the edit history was not committed | Inspect `~/.claude/projects/.../*.jsonl` for earlier `Read`, `Write`, and `Edit` captures, reconstruct the last known good content from those transcript chunks, then move the canonical source back into version-controlled `src/` files instead of keeping it only in generated output |

Practical lessons from the recovery session:

- treat transcript recovery as emergency fallback, not normal workflow
- never keep the only editable copy of a UI or config inside `dist/` or other generated directories
- after recovery, update build scripts so `src/` is authoritative and generated output is disposable

```bash
# Check system health
openclaw doctor

# Follow real-time logs
openclaw logs --follow

# Check agent bindings
openclaw agents list --bindings

# Verify skills are loaded
ls -la ~/.openclaw/skills/
ls -la ~/.openclaw/workspace-<agent>/skills/

# Test network connectivity
nc -zv server-ip port-number

# Check file integrity after writing
wc -l filename.md && tail -3 filename.md
```
