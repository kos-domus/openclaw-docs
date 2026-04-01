---
title: Google Workspace CLI (GWS) Integration
slug: google-workspace-cli
category: guides
tags:
- gws
- google-workspace
- gmail
- drive
- calendar
- skills
sources:
- sessions/2026-03-19-google-workspace-cli-gws-integration.md
- sessions/2026-03-25-setup-complete-drive-skills-testing.md
last_updated: '2026-03-30'
version: 2
---

# Google Workspace CLI (GWS) Integration

GWS is Google's official CLI for interacting with Google Workspace services. It's designed for AI agent integration and uses dynamic API discovery — it reads Google's APIs at runtime and updates itself automatically.

## GWS vs GOG: Which to Choose?

| | GWS | GOG |
|---|---|---|
| Author | Google | Peter Steinberger (OpenClaw creator) |
| Language | Node.js | Go |
| API discovery | Dynamic (reads APIs at runtime) | Static |
| Focus | AI agent integration | CLI scripting |
| OpenClaw integration | Native (skills repo) | Via ClawHub |
| Status | "Developer sample" (not officially supported) | Community-maintained |

**Recommendation**: GWS for new setups — it has native OpenClaw skills and auto-updates when Google adds new API endpoints.

## Installation

```bash
# Prerequisite: Node.js 18+
node --version

# Install GWS globally
npm install -g @googleworkspace/cli
gws --version
```

## Authentication

```bash
# IMPORTANT: Always specify services with -s
# Testing-mode apps are limited to ~25 OAuth scopes
# A generic login requests 85+ scopes and will fail
gws auth login -s drive,gmail,calendar,sheets
```

When prompted with **"Google hasn't verified this app"**, click **Advanced → Go to (app name) (unsafe)**. This is expected for personal-use apps in testing mode.

### Headless Server Authentication

If your server has no browser:

```bash
# On a machine with a browser
gws auth export --unmasked > credentials.json

# Copy to the server
scp credentials.json user@server:~/.config/gws/credentials.json
```

## Installing Skills for OpenClaw

GWS comes with a skills repository that you link into OpenClaw via symlinks:

```bash
# Clone the GWS skills repo
git clone https://github.com/googleworkspace/cli.git ~/gws-cli

# Create the skills directory
mkdir -p ~/.openclaw/skills

# REQUIRED: Base shared skill (dependency for all others)
ln -s ~/gws-cli/skills/gws-shared ~/.openclaw/skills/gws-shared
```

### Available Skills

| Category | Skills |
|----------|--------|
| **Gmail** | `gws-gmail`, `gws-gmail-send`, `gws-gmail-triage`, `gws-gmail-watch` |
| **Calendar** | `gws-calendar`, `gws-calendar-agenda`, `gws-calendar-insert` |
| **Drive** | `gws-drive`, `gws-drive-upload` |
| **Docs** | `gws-docs`, `gws-docs-write` |
| **Sheets** | `gws-sheets`, `gws-sheets-append`, `gws-sheets-read` |
| **Workflow** | `gws-workflow`, `gws-workflow-standup-report`, `gws-workflow-meeting-prep`, `gws-workflow-weekly-digest`, `gws-workflow-email-to-task` |

### Link Skills Selectively

Not every agent needs every skill. Follow the principle of least privilege:

```bash
# Family agent: calendar + drive + email
ln -s ~/gws-cli/skills/gws-calendar ~/.openclaw/skills/gws-calendar
ln -s ~/gws-cli/skills/gws-drive ~/.openclaw/skills/gws-drive
ln -s ~/gws-cli/skills/gws-gmail ~/.openclaw/skills/gws-gmail
ln -s ~/gws-cli/skills/gws-gmail-send ~/.openclaw/skills/gws-gmail-send

# Work agent: docs + sheets + drive
ln -s ~/gws-cli/skills/gws-docs ~/.openclaw/skills/gws-docs
ln -s ~/gws-cli/skills/gws-drive ~/.openclaw/skills/gws-drive
ln -s ~/gws-cli/skills/gws-sheets ~/.openclaw/skills/gws-sheets
```

### Why Symlinks?

Symlinks auto-update when you pull the latest GWS repo:

```bash
cd ~/gws-cli && git pull
# All linked skills are immediately updated
```

## Enable Auto-Loading

Add to `openclaw.json`:

```json
{
  "skills": {
    "load": {
      "watch": true
    }
  }
}
```

Then restart: `openclaw gateway restart`

## Quick Reference Commands

```bash
# Gmail
gws gmail +triage              # Unread email summary

# Calendar
gws calendar +agenda           # Today's events

# Drive
gws drive files list --params '{"pageSize": 5}'
gws drive files list --params '{"q": "'FOLDER_ID' in parents"}'

# Sheets
gws sheets values get --params '{"spreadsheetId": "ID", "range": "Sheet1!A:Z"}'
```

## Key Considerations

- **`gws-shared` is mandatory** — it's the base dependency for all other GWS skills
- **Skills in `~/.openclaw/skills/` are global** — all agents on the machine can use them. For per-agent skills, place them in `workspace/skills/`
- **Google may suspend bot accounts** — automated usage patterns can trigger anti-bot detection. Always use a dedicated account
- **GWS is a "developer sample"** — Google doesn't officially support it, but it has become the de facto standard for AI agent integration

## What's Next

- [GWS CLI Reference](../reference/gws-cli-reference.md) — Complete command reference
- [Google Drive Integration](google-drive-integration.md) — OAuth and account setup
