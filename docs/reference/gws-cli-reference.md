---
title: "GWS CLI Command Reference"
slug: "gws-cli-reference"
category: "reference"
tags: ["gws", "google-workspace", "cli", "reference"]
sources: ["sessions/2026-03-19-google-workspace-cli-gws-integration.md", "sessions/2026-03-25-setup-complete-drive-skills-testing.md"]
last_updated: "2026-03-29"
version: 1
---

# GWS CLI Command Reference

Complete command reference for the Google Workspace CLI (`gws`), Google's Node.js CLI for AI agent integration with Google Workspace services.

## Installation

```bash
npm install -g @googleworkspace/cli
gws --version
```

Requires Node.js 18+.

## Authentication

```bash
# Login with specific services (REQUIRED for testing-mode apps)
gws auth login -s drive,gmail,calendar,sheets

# Export credentials for transfer to headless server
gws auth export --unmasked > credentials.json

# Import on headless server
cp credentials.json ~/.config/gws/credentials.json
```

> ⚠️ **Always use `-s` to specify services.** Without it, `gws auth login` requests 85+ OAuth scopes, which exceeds the ~25 scope limit for testing-mode apps.

## Drive Commands

```bash
# List files (paginated)
gws drive files list --params '{"pageSize": 10}'

# List files in a specific folder
gws drive files list --params '{"q": "'FOLDER_ID' in parents", "fields": "files(id, name, mimeType)"}'

# Search by name
gws drive files list --params '{"q": "name = 'filename.json' and 'FOLDER_ID' in parents"}'

# Download a file (output to stdout)
gws drive files get --params '{"fileId": "FILE_ID", "alt": "media"}'

# Create a folder
gws drive files create --json '{"name": "New Folder", "mimeType": "application/vnd.google-apps.folder", "parents": ["PARENT_ID"]}'

# Upload a file
gws drive files create --json '{"name": "data.json", "parents": ["FOLDER_ID"]}' \
  --upload "data.json" --upload-content-type "application/json"

# Update an existing file
gws drive files update --params '{"fileId": "FILE_ID"}' \
  --upload "updated.json" --upload-content-type "application/json"
```

> ⚠️ **`--upload` requires relative paths from the current working directory.** Absolute paths or paths outside the cwd will fail. Always `cd` into the project directory first.

## Gmail Commands

```bash
# Email triage (unread summary)
gws gmail +triage

# List messages
gws gmail users.messages list --params '{"userId": "me", "maxResults": 10}'

# Read a message
gws gmail users.messages get --params '{"userId": "me", "id": "MESSAGE_ID"}'
```

## Calendar Commands

```bash
# Today's agenda
gws calendar +agenda

# List events
gws calendar events list --params '{"calendarId": "primary", "timeMin": "2026-03-29T00:00:00Z", "maxResults": 10}'

# Create an event
gws calendar events insert --params '{"calendarId": "primary"}' \
  --json '{"summary": "Meeting", "start": {"dateTime": "2026-03-30T10:00:00+01:00"}, "end": {"dateTime": "2026-03-30T11:00:00+01:00"}}'
```

## Sheets Commands

```bash
# Read a range
gws sheets values get --params '{"spreadsheetId": "SHEET_ID", "range": "Sheet1!A:Z"}'

# Append rows
gws sheets values append --params '{"spreadsheetId": "SHEET_ID", "range": "Sheet1!A:A", "valueInputOption": "USER_ENTERED"}' \
  --json '{"values": [["Col1", "Col2", "Col3"]]}'
```

## Key Behaviors

| Behavior | Details |
|----------|---------|
| **Dynamic API discovery** | GWS reads Google APIs at runtime — no static command list |
| **Auto-update** | When Google adds new endpoints, GWS discovers them automatically |
| **Skills via symlink** | Link skills from `~/gws-cli/skills/` to agent workspaces |
| **`gws-shared` required** | Base dependency for all other GWS skills |
| **Testing mode limits** | ~25 OAuth scopes, 100 test users max |

## Skills Available

| Skill | Alias Commands |
|-------|---------------|
| `gws-gmail` | Core Gmail operations |
| `gws-gmail-send` | Send emails |
| `gws-gmail-triage` | `+triage` — unread email summary |
| `gws-gmail-watch` | Monitor inbox |
| `gws-calendar` | Core Calendar operations |
| `gws-calendar-agenda` | `+agenda` — today's events |
| `gws-calendar-insert` | Create events |
| `gws-drive` | Core Drive operations |
| `gws-drive-upload` | Upload files |
| `gws-docs` | Read Google Docs |
| `gws-docs-write` | Create/edit Google Docs |
| `gws-sheets` | Core Sheets operations |
| `gws-sheets-append` | Append rows |
| `gws-sheets-read` | Read ranges |
| `gws-workflow` | Multi-step workflow automation |
