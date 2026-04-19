---
title: Integration Issues
slug: integration-issues
category: troubleshooting
tags:
- troubleshooting
- google
- whatsapp
- telegram
- 1password
sources:
- sessions/2026-03-16-google-drive-integration.md
- sessions/2026-03-18-whatsapp-integration.md
- sessions/2026-03-24-telegram-supergroup-binding-fix.md
- sessions/2026-03-28-1password-full-integration-2.md
- sessions/2026-04-17-spesify-security-ux-stores-launch.md
- sessions/2026-04-17-spesify-sidebar-profile-search-chaininfo-2.md
- sessions/2026-04-18-spesify-nearby-store-drilldown-watchlist.md
last_updated: '2026-04-18'
version: 3
---

# Integration Issues

Troubleshooting guide for third-party service integrations with OpenClaw.

## Google Workspace

### OAuth on Headless Servers

**Problem**: The OAuth flow requires a browser, but headless servers have none.

**Solution**: Create an SSH tunnel from a machine with a browser:

```bash
ssh -L 8080:localhost:8080 user@headless-server
```

Then complete OAuth at `http://localhost:8080` in your local browser.

### Refresh Token Management

OAuth refresh tokens issued to bot accounts are **permanent** (they don't expire unless manually revoked). On an always-on server, this means:

- No need for periodic re-authentication
- **Emergency kill switch**: Revoke at [myaccount.google.com/permissions](https://myaccount.google.com/permissions)
- If the token stops working, re-run the OAuth flow

### Google Account Suspension Risk

Google may suspend accounts that exhibit automated usage patterns. To mitigate:
- Use a dedicated bot account (not personal)
- Avoid rapid-fire API calls (respect rate limits)
- Don't send mass emails from the bot account
- Monitor for "unusual activity" alerts from Google

### Drive File Operations from Wrong Directory

GWS's `--upload` flag validates that the file path is **relative to the current working directory**. Using absolute paths or paths outside the cwd will fail silently.

**Fix**: Always `cd` into the directory containing the files before running GWS upload commands.

## WhatsApp (Baileys)

### Session Expiration

WhatsApp Web sessions expire after **14 days offline**. If the phone with the bot's SIM loses internet:

1. The bot will stop responding
2. You'll need physical access to the phone to re-scan a QR code
3. Run `openclaw channels login --channel whatsapp --qr-terminal`

**Prevention**: Keep the phone charged and connected to WiFi.

### Group ID Discovery

WhatsApp does not expose Group IDs in the app interface. The suggested "Advanced info" menu path in some tutorials **does not exist**.

**Reliable methods**:
1. Send a message in the group, then `grep "@g.us" /tmp/openclaw/openclaw.log`
2. Use `openclaw channels whatsapp groups` command

### Multi-Agent Limitations

WhatsApp does not cleanly support multiple agents on a single phone number. Symptoms:
- Agents intercept each other's messages
- Binding conflicts ("already claimed by another agent")

**Solution**: Dedicate WhatsApp to a single agent context. Use Telegram for multi-agent setups.

## Telegram

### Bot Privacy Mode

Bots in Telegram groups cannot see messages by default. You **must** disable privacy mode:

```
BotFather → /setprivacy → select bot → Disable
```

This must be done for **every bot** that needs to read group messages. It's the #1 missed step in Telegram integration.

### Topic/Thread Routing

OpenClaw's binding schema **does not support `threadId`** for Telegram supergroup topics. Adding `threadId` to the peer object causes an "Invalid input" validation error.

**Workaround**: Bind the entire supergroup to a single orchestrator agent. The orchestrator then routes internally based on message context.

### Thread ID Discovery

The gateway's polling consumes Telegram updates, making them unreadable via the Bot API. To discover topic thread IDs:

1. Stop the gateway: `systemctl --user stop openclaw-gateway.service`
2. Send a message in each topic
3. Query the Bot API: `curl -s "https://api.telegram.org/bot${TOKEN}/getUpdates"`
4. Parse the `message_thread_id` field
5. Restart the gateway

**Note**: The General/default topic has `threadId=None`.

### Multiple Bot Accounts

Each Telegram bot is an independent account with its own token. For a multi-context setup:

| Context | Separate Bot? | Why |
|---------|--------------|-----|
| Personal DM | Yes | Clean separation |
| Work framework | Yes | Topic-based supergroup |
| Side projects | Yes or shared | Depends on scale |

## 1Password

### Desktop App vs CLI Communication

On Linux, the 1Password desktop app and CLI communicate via a Unix socket at `~/.1password/agent.sock`. This socket is **never created** if:

- Desktop app installed via snap, CLI via apt (sandbox isolation)
- Both installed via apt but the "Developer Environment mount handler" fails
- User is not in the `onepassword-cli` group

**Recommended solution**: **Skip desktop app integration entirely.** Use a service account instead — it's more reliable, works headless, and doesn't require a running desktop app.

### Service Account Quirks

- Tokens don't expire by default (format: `ops_...`)
- Require `--vault` flag on `op item get` commands
- `op read` with `op://Vault/Item/field` syntax works without `--vault`
- Session tokens from `op signin` are process-bound — they cannot be shared between terminal sessions

### Secrets Loader Script

When sourcing `op-env.sh` from `.bashrc`:
- Ensure `OP_SERVICE_ACCOUNT_TOKEN` is set **before** any `op read` calls
- Avoid `set -euo pipefail` in the sourced script (causes silent failures)
- Expect 3–5 second startup delay for multiple `op read` calls


### Mini App auth works once, then every later request fails

**Cause**: replay protection is blocking legitimate reuse of Telegram `initData` inside the same Mini App session. Telegram signs one payload that can stay active for the whole session.

**Fix**: verify the Telegram signature, but do not treat repeated use of the same valid payload as an attack inside that active session. Also use a realistic `auth_date` window for long-lived Mini App sessions.

### Mini App preferences keep resetting

**Cause**: preferences are tied only to strict Telegram-authenticated API writes, so any auth hiccup wipes the user experience.

**Fix**: keep dual persistence. Save server-side when auth is available, but also keep a local fallback such as `localStorage` for UI continuity.

### GPS or location-based UI hangs forever inside Telegram

**Cause**: Telegram WebView location flows can stall, and plain browser geolocation may be unavailable or denied.

**Fix**: try Telegram `LocationManager` first, add an explicit timeout, then fall back to `navigator.geolocation` for browser testing. Always clear loading state on both success and failure paths.

### CSP suddenly breaks every `onclick` handler in the web app

**Cause**: Helmet CSP defaults can block inline script attributes with `script-src-attr 'none'`.

**Fix**: either remove inline handlers entirely or explicitly account for them during the transition. The processed sessions used delegated event listeners where possible and only relaxed `scriptSrcAttr` when needed to unblock the UI.

### A service works manually but crashes in user-mode systemd after hardening

**Cause**: some hardening directives that look reasonable on paper are not compatible with a user-mode systemd service or with the runtime stack you are launching.

**Fix**: keep the hardening set modest and verify each directive in the real target mode. `NoNewPrivileges=yes` and `PrivateTmp=yes` held up, while more aggressive directives caused startup failures in the processed sessions.

### Pipeline wrapper cannot run follow-up steps after the main script

**Cause**: the wrapper uses `exec` on the main process, so the shell never reaches the later commands.

**Fix**: call the main process normally, then chain the follow-up step explicitly. This pattern mattered for post-pipeline notifications and other best-effort automation tasks.

## General Integration Debugging

1. **Check logs first**: `openclaw logs --follow`
2. **Test the service independently**: Verify GWS, Telegram Bot API, etc. work outside OpenClaw
3. **Restart the gateway**: `openclaw gateway restart` after any config change
4. **Verify bindings**: `openclaw agents list --bindings`
5. **Check permissions**: `ls -la` on credential files, skill symlinks
6. **Test connectivity**: `nc -zv host port` for network issues
