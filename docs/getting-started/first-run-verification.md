---
title: "First Run Verification: Testing Your Full Setup"
slug: "first-run-verification"
category: "getting-started"
tags: ["setup", "verification", "testing", "multi-agent"]
sources: ["sessions/2026-03-25-setup-complete-drive-skills-testing.md"]
last_updated: "2026-03-29"
version: 1
---

# First Run Verification: Testing Your Full Setup

After installing OpenClaw, configuring authentication, and setting up channels, this guide walks you through verifying that everything works end-to-end.

## Prerequisites

- OpenClaw installed and gateway running (`openclaw status` shows active)
- At least one AI provider configured (Anthropic, OpenAI, etc.)
- At least one channel configured (WhatsApp, Telegram, etc.)

## Step 1: Verify Gateway and Agents

```bash
# Check gateway status
openclaw status

# List registered agents with their bindings
openclaw agents list --bindings
```

You should see your agents listed with their workspace paths and channel bindings. If no agents appear, you need to register them first with `openclaw agents add`.

## Step 2: Test Channel Connectivity

### WhatsApp

Send a message in the bound WhatsApp group. Check the logs:

```bash
openclaw logs --follow
```

You should see the incoming message logged, the agent processing it, and the response being sent back.

### Telegram

Send a message to the bot's DM or in the bound group. For supergroups with topics, send in the specific topic that's bound to an agent.

**Important**: For Telegram group bots, privacy mode must be **disabled** in BotFather:
```
BotFather → /setprivacy → select bot → Disable
```

Without this, bots cannot see group messages.

## Step 3: Test Google Workspace Integration

If you've configured GWS skills, test each service:

```bash
# Test Gmail access
gws gmail +triage

# Test Calendar access
gws calendar +agenda

# Test Drive access
gws drive files list --params '{"pageSize": 5}'
```

Then test via the chat channel — send messages like:
- "How many unread emails do I have?"
- "What's on the calendar today?"
- "List the files in my Drive"

## Step 4: Verify Skill Loading

```bash
# List loaded skills
ls -la ~/.openclaw/skills/

# Check workspace-specific skills
ls -la ~/.openclaw/workspace-<agent>/skills/
```

Skills are loaded via symlinks. If a skill isn't being invoked by the agent, check:
1. The symlink exists and points to a valid directory
2. `gws-shared` is linked (required base dependency for all GWS skills)
3. The gateway was restarted after adding skills: `openclaw gateway restart`

## Step 5: Multi-Agent Routing Verification

If you have multiple agents, verify that messages route to the correct agent:

| Test | Expected Result |
|------|----------------|
| Message in WhatsApp group "Family" | Handled by family agent (e.g., Kai) |
| Message in Telegram DM to Bot A | Handled by agent bound to Bot A |
| Message in Telegram supergroup topic | Handled by the agent bound to that group |

**Common routing issue**: If the wrong agent responds, check the binding configuration in `openclaw.json`. The most common mistake is using `accountId` where `peer` should be used:

```json
// WRONG — puts group ID in accountId
{ "match": { "channel": "whatsapp", "accountId": "<group_jid>" } }

// CORRECT — uses peer for group filtering
{ "match": { "channel": "whatsapp", "accountId": "default", "peer": { "kind": "group", "id": "<group_jid>" } } }
```

## Step 6: Test Selective Tool Access

Each agent should only have access to the tools it needs. Verify by asking agents to perform operations outside their scope — they should refuse or report inability.

Key tool policies to verify:
- Family/personal agents: **no `exec`** (prevents command execution on sensitive data)
- Public-facing agents: **sandbox mode `always`** (isolated sessions)
- Admin agents: full access but with `exec_ask` enabled (confirmation prompt before shell commands)

## Verification Checklist

- [ ] `openclaw status` shows gateway active
- [ ] All agents registered (`openclaw agents list`)
- [ ] Each channel delivers messages to the correct agent
- [ ] GWS commands return data (if configured)
- [ ] Skills loaded in correct workspaces
- [ ] Agents respond via chat channel
- [ ] Multi-agent routing works (each group/DM hits the right agent)
- [ ] Tool access restrictions enforced

## What's Next

- [Google Workspace CLI Guide](../guides/google-workspace-cli.md) — Deep dive into GWS integration
- [Multi-Agent Architecture](../concepts/multi-agent-architecture.md) — Understanding the agent hierarchy
- [Common Errors](../troubleshooting/common-errors.md) — If something isn't working
