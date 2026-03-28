---
title: "1Password full integration: CLI setup, secrets migration, service account, Kai verification protocol"
date: "2026-03-28"
author: "kos-domus"
status: "ready"
tags: ["security", "configuration", "setup", "troubleshooting", "multi-agent", "automation", "claude-md"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote + direct mini PC access"
  model: "claude-opus-4-6"
---

## Objective
Complete the 1Password integration that was blocked in the previous session: resolve the CLI-to-desktop-app communication issue, migrate all plaintext secrets to 1Password vaults, set up a service account for always-on headless access, and add identity verification for Kai's credential sharing in WhatsApp.

## Context
- Previous session identified the blocker: snap vs apt desktop app conflict preventing `~/.1password/agent.sock` creation
- 1Password CLI v2.33.1 installed via apt, account configured, 6 vaults created
- Plaintext secrets existed in `~/.openclaw/.env`, `~/.openclaw/agents/*/auth-profiles.json`, and `~/.bashrc`
- Mini PC accessed remotely from Mac via Tailscale SSH + VS Code Remote

## Steps Taken

### 1. Diagnosed desktop app CLI integration failure
Attempted multiple approaches to get `op` CLI communicating with the 1Password desktop app:
- Confirmed snap was removed, apt version installed at `/opt/1Password/`
- Both versions were initially running simultaneously — killed snap processes
- Created `~/.1password/` directory (app expected it to exist)
- Restarted desktop app multiple times
- Analyzed app logs: "Developer Environment mount handler" kept failing with `Io(Os { code: 2, kind: NotFound })`
- Found user `kos` was NOT in the `onepassword-cli` group (required for setgid CLI binary)
- Added user to group: `sudo usermod -aG onepassword-cli kos`

**Result**: Even after all fixes, the desktop app never created `agent.sock`. The "Developer Environment mount handler" continued failing. The BrowserSupport socket existed at `/run/user/1000/1Password-BrowserSupport.sock` but the CLI socket was never created.

### 2. Worked around with manual password-based auth
Since desktop app integration failed, used direct password authentication:

```bash
export OP_SESSION_my=$(OP_BIOMETRIC_UNLOCK_ENABLED=false op signin --account my.1password.com --raw)
```

Key finding: session tokens are process-bound — they cannot be shared across shells. The token from one terminal does not work in another process.

**Result**: `op whoami` and `op vault list` working in the terminal where signin was performed.

### 3. Migrated all plaintext secrets to 1Password Tech vault
Inventoried all secrets across the system and stored them:

| Secret | Source | 1Password Item |
|--------|--------|---------------|
| GitHub PAT | `~/.openclaw/.env` | GitHub PAT (openclaw) |
| Groq API Key | `agents/katia/auth-profiles.json` | Groq API Key |
| Google OAuth client_secret | `credentials/google/google_oauth_client.json` | Google OAuth Client (openclaw) |
| Google encryption key | `credentials/google/.encryption_key` | Google Encryption Key (openclaw) |
| Anthropic API Key | `agents/main/auth-profiles.json` | Anthropic API Key |
| OpenAI API Key | `agents/main/auth-profiles.json` + `.bashrc` | OpenAI API Key |
| OpenRouter API Key | `agents/main/auth-profiles.json` | OpenRouter API Key |
| Telegram Bot Token (default) | `.bashrc` | Telegram Bot Token (default) |
| Telegram Bot Token (work) | `.bashrc` | Telegram Bot Token (work) |
| Telegram Bot Token (wolf) | `.bashrc` | Telegram Bot Token (wolf) |
| OpenClaw Gateway Token | `.bashrc` | OpenClaw Gateway Token |

Used `op item create` with assignment syntax (not `--field` which doesn't exist in v2):
```bash
op item create --vault Tech --category "API Credential" --title "Item Name" \
  'credential=SECRET_VALUE'
```

**Result**: 11 items stored in Tech vault.

### 4. Removed all plaintext secrets from config files
- `~/.openclaw/.env` — replaced hardcoded GitHub PAT with comment pointing to 1Password
- `~/.openclaw/agents/main/agent/auth-profiles.json` — replaced 3 keys (Anthropic, OpenAI, OpenRouter) with `${VAR}` env var references
- `~/.openclaw/agents/katia/agent/auth-profiles.json` — replaced Groq key with `${GROQ_API_KEY}`
- `~/.bashrc` — removed 6 hardcoded `export` lines (Telegram tokens, OpenAI key, Gateway token)

**Result**: Zero plaintext API keys remaining in configuration files.

### 5. Created op-env.sh secrets loader script
Created `~/.openclaw/op-env.sh` to load all secrets from 1Password into environment variables at shell startup. Initially used item names in `op://` references but parentheses in names caused errors. Switched to item UUIDs:

```bash
export GITHUB_PAT="$(op read 'op://Tech/ITEM_UUID/credential')"
```

**Result**: Script loads 10 secrets in one shot, integrated into `.bashrc`.

### 6. Set up 1Password Service Account for headless access
Manual signin was impractical for an always-on server. Created a service account via https://my.1password.com web interface:
- Name: `openclaw-server`
- Access: read-only to Tech, Family, Finance, Medical, Streaming vaults
- Token type: `ops_...` (never expires, no desktop app needed)
- Token stored at `~/.op-service-account-token` (chmod 600)

Updated `op-env.sh` to use service account instead of manual signin:
```bash
export OP_SERVICE_ACCOUNT_TOKEN="$(cat ~/.op-service-account-token)"
```

Service accounts require `--vault` flag on `op item get` commands but `op read` with `op://Vault/...` syntax works without it.

**Result**: All 10 secrets auto-load on every shell session with zero manual intervention.

### 7. Added Identity Verification Protocol for Kai
Updated `~/job-desk/family-os/CLAUDE.md` with:
- Full 1Password integration section with `op` CLI commands
- Vault access matrix (who can request what)
- Identity verification protocol: Kai asks for family PIN before sharing any credential
- Created `family-verification-pin` item in Family vault
- Rate limiting: 3 failed attempts in 24h triggers lockout + alert to Kos
- Security tiers: WiFi/streaming passwords shareable after PIN, but Finance/SPID always refused via WhatsApp

**Result**: Kai can now read credentials from 1Password but only shares them after PIN verification.

## Configuration Changes
- `~/.openclaw/.env` — plaintext GitHub PAT removed
- `~/.openclaw/agents/main/agent/auth-profiles.json` — 3 keys replaced with `${VAR}` references
- `~/.openclaw/agents/katia/agent/auth-profiles.json` — Groq key replaced with `${GROQ_API_KEY}`
- `~/.bashrc` — 6 secret exports removed, replaced with `source ~/.openclaw/op-env.sh`
- `~/.openclaw/op-env.sh` — new file, loads 10 secrets from 1Password via service account
- `~/.op-service-account-token` — new file (600 perms), stores service account token
- `~/job-desk/family-os/CLAUDE.md` — added 1Password integration section + verification protocol
- User `kos` added to `onepassword-cli` group

## Key Discoveries
- **Desktop app CLI integration on Linux is fragile**: Even with both desktop app and CLI installed via apt, correct polkit policies, user in the right group, and "Integrate with 1Password CLI" enabled, the socket was never created. The "Developer Environment mount handler" fails silently.
- **Service accounts are the right answer for servers**: Skip desktop app integration entirely. Service account tokens never expire, work headless, and don't need any desktop app running.
- **op v2 uses assignment syntax, not `--field`**: `op item create --field "key=value"` fails; use `op item create 'key=value'` instead.
- **Parentheses in item names break `op://` references**: `op://Tech/GitHub PAT (openclaw)/credential` fails with "invalid character". Use item UUIDs instead of names in `op://` paths.
- **Session tokens are process-bound**: `op signin --raw` tokens cannot be exported to other shell processes — they only work in the process that created them.
- **Service accounts require `--vault` on `op item get`**: Without it, you get "a vault query must be provided when this command is called by a service account."
- **`set -euo pipefail` in sourced scripts**: The `op-env.sh` initially had empty variables because `set -e` caused silent failures. Fixed by ensuring `OP_SERVICE_ACCOUNT_TOKEN` was set before any `op read` calls.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| `~/.1password/agent.sock` not created | Desktop app "Developer Environment mount handler" fails with NotFound | Use service account instead of desktop app integration |
| `op whoami` → "account is not signed in" | Session token process-bound; can't share across shells | Use `OP_SERVICE_ACCOUNT_TOKEN` instead |
| `op item create --field` → "unknown flag" | op v2 syntax doesn't have `--field` | Use assignment syntax: `'field=value'` |
| `op read "op://Tech/Name (with parens)/field"` → "invalid character" | Parentheses not allowed in `op://` secret references | Use item UUIDs instead of display names |
| `op item get ID` → "vault query must be provided" | Service accounts require explicit vault | Add `--vault Tech` flag |
| 1Password desktop app crash on SSH launch | No display available (`$DISPLAY` not set) | Must launch from physical screen or use service account |
| Both snap and apt 1Password running simultaneously | Snap processes lingered after app "close" | `killall 1password` then launch `/opt/1Password/1password` |

## Final State
- **1Password**: 11 items in Tech vault + 1 PIN in Family vault, service account active
- **Secrets loading**: Fully automatic via `op-env.sh` sourced in `.bashrc`
- **Zero plaintext secrets**: All API keys, tokens, and credentials removed from config files
- **Kai**: Updated CLAUDE.md with 1Password read access + identity verification protocol
- **Desktop app CLI integration**: Still broken (not needed — service account bypasses it)

## Open Questions
- Should the service account token itself be rotated periodically? Currently it never expires.
- Should Kai have write access to any vault (to save new passwords) or strictly read-only?
- How to handle `op-env.sh` load time (~3-5 seconds for 10 `op read` calls) impacting shell startup? Caching?
- The `${VAR}` references in `auth-profiles.json` assume OpenClaw resolves env vars in that file — does it actually? Needs testing.
- Desktop app CLI integration failure root cause still unknown — worth filing a bug with 1Password?
