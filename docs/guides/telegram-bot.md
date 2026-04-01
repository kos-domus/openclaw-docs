---
title: Telegram Bot Setup and Supergroup Topics
slug: telegram-bot
category: guides
tags:
- telegram
- channels
- multi-agent
- supergroup
- troubleshooting
sources:
- sessions/2026-03-24-telegram-supergroup-binding-fix.md
last_updated: '2026-03-30'
version: 2
---

# Telegram Bot Setup and Supergroup Topics

This guide covers setting up Telegram bots with OpenClaw, including supergroup configuration with topic-based routing for multi-agent setups.

## Prerequisites

- OpenClaw installed and running
- A Telegram bot token from [@BotFather](https://t.me/BotFather)

## Step 1: Create a Bot via BotFather

1. Open [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the prompts
3. Save the bot token (format: `123456:ABC-DEF1234ghIkl-...`)

### Critical: Disable Privacy Mode

For bots that need to read group messages:

```
BotFather → /setprivacy → select your bot → Disable
```

**Without this, your bot cannot see any group messages.** This is the most commonly missed step.

## Step 2: Configure the Bot in OpenClaw

Add the Telegram account to your configuration and bind an agent:

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "my-bot": {
          "token": "your-bot-token",
          "groupPolicy": "allowlist",
          "allowGroups": ["GROUP_CHAT_ID"]
        }
      }
    }
  },
  "bindings": [
    {
      "type": "route",
      "agentId": "your-agent",
      "match": {
        "channel": "telegram",
        "accountId": "my-bot",
        "peer": { "kind": "group", "id": "GROUP_CHAT_ID" }
      }
    }
  ]
}
```

## Step 3: Setting Up Supergroups with Topics

Telegram supergroups support **topics** (forum mode), which let you organize conversations by subject. This is useful for routing different topics to different agents.

### Create the Supergroup

1. Create a group in Telegram
2. Enable **Forum/Topics** mode in group settings
3. Promote your bot to admin
4. Create topics for each agent/function

### Find Topic Thread IDs

OpenClaw's gateway consumes Telegram updates via polling, so you can't simply read them from the Bot API while the gateway is running. Here's the workaround:

```bash
# 1. Stop the gateway temporarily
systemctl --user stop openclaw-gateway.service

# 2. Send a message in each topic (so there are updates to read)

# 3. Read the updates directly from the Bot API
curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?limit=50" | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for u in data.get('result', []):
    msg = u.get('message', {})
    thread = msg.get('message_thread_id')
    text = msg.get('text', '')[:40]
    chat = msg.get('chat', {}).get('id')
    print(f'chatId={chat} threadId={thread} text={text!r}')
"

# 4. Restart the gateway
systemctl --user start openclaw-gateway.service
```

The **General** topic has `threadId=None` — it's the default thread of the supergroup.

### Topic Routing Limitation

> ⚠️ **OpenClaw does not natively support `threadId` in binding schemas.** Attempting to add a `threadId` field to the peer object results in an "Invalid input" validation error.

**Workaround**: Bind the entire supergroup to a single orchestrator agent, which then delegates internally to sub-agents:

```json
{
  "type": "route",
  "agentId": "orchestrator",
  "match": {
    "channel": "telegram",
    "accountId": "work-bot",
    "peer": { "kind": "group", "id": "SUPERGROUP_CHAT_ID" }
  }
}
```

## Multiple Bots for Multiple Contexts

Since topic-based routing isn't natively supported, a practical pattern is using separate bot accounts for different contexts:

| Bot | Context | Binding |
|-----|---------|---------|
| Bot A | Personal assistant | DM only |
| Bot B | Work framework | Supergroup with topics |
| Bot C | Side projects | Separate group |

Each bot is an independent Telegram account with its own token.

## Common Issues

| Problem | Cause | Solution |
|---------|-------|---------|
| Bot doesn't respond in groups | Privacy mode enabled | Disable in BotFather: `/setprivacy → Disable` |
| "not-allowed" in logs | Group not in allowlist | Add the group chat ID to `allowGroups` |
| Wrong agent responds | Binding overlap | Remove generic bindings before adding specific ones |
| "Skipped bindings already claimed" | Another agent's broad binding catches the message first | Make bindings more specific or reorder them |
| Can't read topic thread IDs | Gateway consuming updates | Stop gateway first, then query Bot API directly |

## What's Next

- [WhatsApp Bot Setup](whatsapp-bot.md) — Add WhatsApp as a channel
- [Multi-Agent Architecture](../concepts/multi-agent-architecture.md) — Design agent hierarchies
