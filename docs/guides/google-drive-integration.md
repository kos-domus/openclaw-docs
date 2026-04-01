---
title: Google Drive Integration for OpenClaw
slug: google-drive-integration
category: guides
tags:
- google-drive
- oauth
- configuration
- security
sources:
- sessions/2026-03-16-google-drive-integration.md
- sessions/2026-03-17-google-account-setup-for-bot.md
last_updated: '2026-03-30'
version: 2
---

# Google Drive Integration for OpenClaw

This guide walks you through connecting OpenClaw to Google Drive so your agents can create, read, modify, and share files programmatically.

## Prerequisites

- OpenClaw installed and running
- A **dedicated Google account** (never use your personal account)
- Access to [Google Cloud Console](https://console.cloud.google.com)

## Why a Dedicated Account?

If your server is compromised, the attacker gains access to every Google service linked to the authenticated account. A dedicated bot account limits the blast radius — your personal email, photos, and documents stay safe.

You may want to set up a dedicated phone number (e.g., a prepaid eSIM) for account verification.

## Step 1: Create a Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project (e.g., "OpenClaw Bot")
3. Navigate to **APIs & Services → Library**
4. Enable the following APIs:
   - Google Drive API
   - Gmail API
   - Google Calendar API
   - Google Sheets API
   - (Optional) Places API, Geocoding API, Directions API

## Step 2: Configure OAuth Consent Screen

1. Go to **APIs & Services → OAuth consent screen**
2. Select **External** user type
3. Fill in the required fields (app name, support email)
4. Add your bot's email as a **test user**
5. Add OAuth scopes:

| Service | Scope | Access Level |
|---------|-------|-------------|
| Gmail | `https://mail.google.com/` | Full access |
| Calendar | `https://www.googleapis.com/auth/calendar` | Full access |
| Drive | `https://www.googleapis.com/auth/drive` | Full access (not `.readonly`) |

> ⚠️ **Note**: Apps in testing mode have a limit of ~25 OAuth scopes and 100 test users. This is sufficient for personal use.

## Step 3: Create OAuth Credentials

1. Go to **Credentials → Create Credentials → OAuth Client ID**
2. Application type: **Desktop app**
3. Download the `client_secret_*.json` file
4. Transfer it to the server:

```bash
mkdir -p ~/.config/gws
scp client_secret_*.json user@server:~/.config/gws/client_secret.json
```

## Step 4: Authenticate on a Headless Server

On a headless server (no browser), complete OAuth using an SSH tunnel:

```bash
# From your local machine (with a browser), create the tunnel
ssh -L 8080:localhost:8080 user@your-server-ip
```

Then complete the OAuth flow at `http://localhost:8080` in your local browser. The credentials are saved locally on the server.

When you see **"Google hasn't verified this app"**, click **Advanced → Go to (app name) (unsafe)**. This is normal for personal/testing apps.

## Step 5: Secure Your Credentials

### OAuth Refresh Token

The refresh token is the bot's "permanent pass" to your Google account. It does not expire unless manually revoked. Treat it like a password.

To revoke access in an emergency: go to [myaccount.google.com/permissions](https://myaccount.google.com/permissions) and remove the app.

### API Key Restrictions (for Maps/Places)

If you enabled Places or Maps APIs, restrict the API key:

1. Go to **Credentials** → select your API key
2. **Application restrictions** → IP addresses → enter your server's **public** IP
3. **API restrictions** → select only the specific APIs needed

Find your public IP with: `curl ifconfig.me`

> ⚠️ **Common mistake**: Google Cloud sees your **public IP** (router's external address), not your LAN IP (192.168.x.x). The geolocation may show a nearby city — this is normal and does not affect functionality.

### Budget Alerts

Set a budget alert at a low threshold (e.g., $1) in **Billing → Budgets & alerts** to catch unexpected usage.

### Audit Logging

Enable audit logs in IAM to monitor API access patterns.

## Step 6: Test the Connection

Once authenticated, verify access:

```bash
# List Drive files
gws drive files list --params '{"pageSize": 5}'

# Check Gmail
gws gmail +triage

# Check Calendar
gws calendar +agenda
```

## Security Best Practices

- **Dedicated account only** — never authenticate with your personal Google account
- **Enable 2FA** on the bot account (use an authenticator app, not SMS if possible)
- **Set a recovery email** that you control
- **Monitor for suspicious activity** — Google sends security alerts for access from unusual devices/locations
- **Separate API keys from OAuth** — Places/Maps use API keys (more vulnerable to theft), Gmail/Drive/Calendar use OAuth (more secure)
- **Beware of Google account suspension** — automated usage patterns may trigger anti-bot detection. A dedicated account limits the impact if suspended.

## What's Next

- [Google Workspace CLI Reference](google-workspace-cli.md) — Detailed GWS command usage
- [1Password Secrets Management](1password-secrets-management.md) — Secure credential storage
