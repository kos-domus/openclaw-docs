---
title: 1Password Secrets Management for OpenClaw
slug: 1password-secrets-management
category: guides
tags:
- security
- 1password
- secrets
- configuration
sources:
- sessions/2026-03-28-1password-full-integration-2.md
- sessions/2026-03-28-familyos-database-kai-1password.md
- sessions/2026-03-31-subscriptions-expense-tracking-3.md
- sessions/2026-03-30-kos-bootstrap-cron-jobs-1password-gemini.md
- sessions/2026-03-30-kai-fixes-kos-openclaw-monitor-4.md
last_updated: '2026-04-01'
version: 4
---

# 1Password Secrets Management for OpenClaw

This guide covers integrating 1Password as a unified secrets provider for OpenClaw, replacing plaintext credentials scattered across configuration files.

## Why 1Password?

OpenClaw deployments typically accumulate secrets in multiple locations:
- `~/.openclaw/.env` — API keys
- `~/.openclaw/agents/*/auth-profiles.json` — Per-agent credentials
- `~/.bashrc` — Exported tokens

This creates a security liability. 1Password centralizes all secrets with:
- Encrypted storage with zero-knowledge architecture
- Service accounts for headless server access
- Audit logging of every access
- Vault-based access control

## Step 1: Install the CLI

```bash
# Add the 1Password apt repository
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] \
  https://downloads.1password.com/linux/debian/amd64 stable main" | \
  sudo tee /etc/apt/sources.list.d/1password.list

sudo apt update && sudo apt install 1password-cli
```

Verify: `op --version` (should show v2.x)

## Step 2: Create Your Vault Structure

Organize secrets by access level and context:

| Vault | Contents | Access |
|-------|----------|--------|
| Tech | API keys, bot tokens, OAuth credentials | Admin only |
| Family | WiFi, router, school portals | Shared family |
| Utilities | Electricity, gas, internet, insurance | Shared family |
| Finance | Banking, tax portals | Admin only |

**Principle**: Group by who needs access, not by service type.

## Step 3: Migrate Secrets

### Store a secret

```bash
op item create --vault Tech --category "API Credential" \
  --title "Descriptive Name" \
  'credential=YOUR_SECRET_VALUE'
```

> ⚠️ **Syntax note**: `op` v2 uses **assignment syntax** (`'field=value'`), not `--field`. The `--field` flag does not exist in v2 — using it produces an "unknown flag" error.

### Common secret types to migrate

- GitHub Personal Access Token
- AI provider API keys (Anthropic, OpenAI, etc.)
- Telegram bot tokens
- Google OAuth client secret
- Gateway authentication token

### Remove plaintext secrets

After storing in 1Password, replace hardcoded values:
- In `.env` files: replace with comments pointing to 1Password
- In `auth-profiles.json`: replace with `${ENV_VAR}` references
- In `.bashrc`: remove `export SECRET=value` lines

## Step 4: Set Up a Service Account

Manual sign-in doesn't work for always-on servers (session tokens are process-bound and expire). Use a **service account** instead.

1. Go to your 1Password web interface
2. Create a service account with **read-only** access to the required vaults
3. Save the token (format: `ops_...`) securely on the server:

```bash
# Store the token with restricted permissions
echo "YOUR_SERVICE_ACCOUNT_TOKEN" > ~/.op-service-account-token
chmod 600 ~/.op-service-account-token
```

## Step 5: Create a Secrets Loader Script

Create a script that loads all secrets from 1Password into environment variables:

```bash
#!/usr/bin/env bash
# ~/.openclaw/op-env.sh — Loads secrets from 1Password

export OP_SERVICE_ACCOUNT_TOKEN="$(cat ~/.op-service-account-token)"

export GITHUB_PAT="$(op read 'op://Tech/ITEM_UUID/credential')"
export ANTHROPIC_API_KEY="$(op read 'op://Tech/ITEM_UUID/credential')"
export OPENAI_API_KEY="$(op read 'op://Tech/ITEM_UUID/credential')"
# ... additional secrets
```

> ⚠️ **Use item UUIDs, not display names** in `op://` references. Item names containing parentheses or special characters cause "invalid character" errors. UUIDs are always safe.

Add to your shell profile:

```bash
# In ~/.bashrc
source ~/.openclaw/op-env.sh
```

## Gotchas and Troubleshooting

| Issue | Cause | Solution |
|-------|-------|---------|
| `op whoami` → "not signed in" | Session tokens are process-bound | Use a service account instead |
| `op read "op://Vault/Name (parens)/field"` fails | Parentheses in item names | Use item UUIDs instead of display names |
| `op item get ID` → "vault query must be provided" | Service accounts require explicit vault | Use `op://Vault/...` syntax or add `--vault` flag |
| Desktop app doesn't create `agent.sock` | Snap vs apt installation conflict | Install both desktop app and CLI from the same source (apt) |
| `set -euo pipefail` causes silent failures | `op read` fails before token is set | Ensure `OP_SERVICE_ACCOUNT_TOKEN` is exported before any `op read` calls |
| Shell startup is slow (~3-5 seconds) | Multiple `op read` calls at login | Consider caching or lazy loading |

## Desktop App Integration (Linux)

> ⚠️ **Linux desktop app CLI integration is fragile.** Even with correct installation, user groups (`onepassword-cli`), and polkit policies, the `~/.1password/agent.sock` socket may never be created. The "Developer Environment mount handler" fails silently.

**Recommendation**: Skip desktop app integration entirely on servers. Service accounts are more reliable and don't require a running desktop app.

## Security Considerations

- **Never commit the service account token** to any repository
- **Use read-only access** for service accounts unless write access is specifically needed
- **Monitor access logs** in the 1Password web interface
- **Token rotation**: Service account tokens don't expire by default — consider periodic rotation as a best practice
- **Subscription fallback keys**: keep `OPENAI_API_KEY`, `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, and `OPENROUTER_API_KEY` in 1Password only as emergency fallbacks; normal agent traffic should go through OAuth/CLI subscriptions
- **Sensitive credentials over chat**: Banking passwords, SPID credentials, and similar high-value secrets should never be shared via messaging platforms, even with identity verification

## What's Next

- [Remote Access and Tailscale](remote-access.md) — Network security
- [Claude Code Settings](../reference/claude-code-settings.md) — Configuration reference
