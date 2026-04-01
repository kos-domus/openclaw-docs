---
title: Setting Up a WhatsApp Bot with OpenClaw
slug: whatsapp-bot
category: guides
tags:
- whatsapp
- channels
- configuration
- baileys
sources:
- sessions/2026-03-18-whatsapp-integration.md
last_updated: '2026-03-30'
version: 2
---

# Setting Up a WhatsApp Bot with OpenClaw

OpenClaw uses **Baileys** (WhatsApp Web protocol) natively — no Meta Business API, no approval process, no message templates. Your bot is operational in minutes.

## Prerequisites

- OpenClaw installed and running
- A dedicated phone number for the bot (eSIM recommended)
- A phone with WhatsApp installed on that number

## Step 1: Choose Your Approach

| | Baileys (via OpenClaw) | Meta Business API |
|---|---|---|
| Approval needed | No | Yes (Meta review) |
| Message templates | No (send freely) | Yes (pre-approved templates for proactive messages) |
| Setup time | Minutes | Days/weeks |
| Cost | Free | Per-message pricing |
| Best for | Personal bots, small groups | Large-scale business messaging |

For personal and family use, Baileys through OpenClaw is the clear choice.

## Step 2: Connect WhatsApp

```bash
# Option A: Through the onboarding wizard
openclaw onboard

# Option B: Direct channel login (displays QR code in terminal)
openclaw channels login --channel whatsapp --qr-terminal
```

When prompted for "the phone you will message from," enter **your personal phone number** — not the bot's number. This sets up the allowlist.

Scan the QR code using the phone with the bot's eSIM:
**WhatsApp → Settings → Linked Devices → Link a Device**

> ⚠️ **QR codes expire in 1–2 minutes.** Have the phone ready before running the command. If it expires, just re-run the command.

## Step 3: Configure Access Policies

Edit `~/.openclaw/openclaw.json`:

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["+1XXXXXXXXXX"],
      "groupPolicy": "allowlist",
      "autoRead": true,
      "typingIndicator": true
    }
  }
}
```

| Setting | Purpose |
|---------|---------|
| `dmPolicy: "allowlist"` | Only numbers in `allowFrom` can DM the bot |
| `groupPolicy: "allowlist"` | Control who can interact in groups |
| `allowFrom` | Authorized phone numbers (E.164 format: `+1...`) |
| `autoRead` | Mark messages as read automatically |
| `typingIndicator` | Show "typing..." while processing |

## Step 4: Find Your Group ID

WhatsApp Group IDs have the format: `<group_jid>` (e.g. `1234567890-1234567890@g.us`)

The Group ID is **not visible** in the WhatsApp app interface. Here's how to find it:

### Method 1: OpenClaw Logs (Easiest)

1. Send any message in the target group
2. Check the logs:

```bash
tail -f /tmp/openclaw/openclaw.log | grep "@g.us"
```

### Method 2: Grep Existing Logs

```bash
grep -i "g.us" /tmp/openclaw/openclaw.log
```

## Step 5: Bind an Agent to a Group

Once you have the Group ID, configure the binding in `openclaw.json`:

```json
{
  "bindings": [
    {
      "type": "route",
      "agentId": "your-agent-id",
      "match": {
        "channel": "whatsapp",
        "accountId": "default",
        "peer": { "kind": "group", "id": "<group_jid>" }
      }
    }
  ]
}
```

> ⚠️ **Critical binding mistake**: Do **not** put the Group JID in `accountId`. The `accountId` identifies the WhatsApp account (use `"default"`), while `peer` identifies the specific group or contact.

## Important Operational Notes

### Session Expiry

The WhatsApp Web session expires after **14 days offline**. If the phone with the bot's eSIM loses internet for too long, you'll need to re-scan the QR code.

### Dual SIM Considerations

If you plan to use the same eSIM for both WhatsApp and Google 2FA, be aware of potential conflicts. Consider using separate numbers or switching Google 2FA to an authenticator app.

### LLM Hallucinations in Config

OpenClaw's AI agents may sometimes suggest configuration parameters that don't actually exist (e.g., `visibility: "all"`, `pruning: "auto"`). **Always verify suggested config changes against the official documentation** before applying them.

### Multi-Agent on One Number

WhatsApp does not cleanly support multiple agents on a single phone number. If you need multiple agents, use different channels (e.g., WhatsApp for family, Telegram for work).

## What's Next

- [Telegram Bot Setup](telegram-bot.md) — Set up Telegram as an additional channel
- [First Run Verification](../getting-started/first-run-verification.md) — Verify the full setup
