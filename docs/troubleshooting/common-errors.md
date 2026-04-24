---
title: "Common Errors and Solutions"
slug: "common-errors"
category: "troubleshooting"
tags: ["errors", "troubleshooting", "debugging"]
sources: ["sessions/2026-03-12-ubuntu-usb-setup-for-openclaw.md", "sessions/2026-03-13-acemagic-ubuntu-openclaw-install.md", "sessions/2026-03-14-anthropic-auth-apikey-vs-setuptoken.md", "sessions/2026-03-19-google-workspace-cli-gws-integration.md", "sessions/2026-03-23-handson-8agent-setup.md", "sessions/2026-03-28-1password-full-integration-2.md", "sessions/2026-03-30-kos-bootstrap-cron-jobs-1password-gemini.md", "sessions/2026-03-31-openclaw-update-oauth-models-monitor.md", "sessions/2026-03-31-subscriptions-expense-tracking-3.md", "sessions/2026-03-30-kai-new-capabilities-reminders-audio-2.md", "sessions/2026-03-30-kai-reminders-audio-implementation-3.md", "sessions/2026-03-30-kai-fixes-kos-openclaw-monitor-4.md", "sessions/2026-03-31-kai-cron-briefings-waste-calendar-memory-2.md", "sessions/2026-04-01-openclaw-v31-acp-kos-pipeline-kai-mensa.md", "sessions/2026-04-20-openclaw-upgrade-4.15-gateway-restart.md", "sessions/2026-04-23-spesify-major-rebuild-fixes.md"]
last_updated: "2026-04-24"
version: 6
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
