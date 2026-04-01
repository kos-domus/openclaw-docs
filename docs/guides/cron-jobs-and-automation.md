---
title: Cron Jobs, Sub-Agents, and Automation
slug: cron-jobs-and-automation
category: guides
tags:
- cron
- automation
- sub-agents
- voice
- tts
sources:
- sessions/2026-03-26-family-bot-features-cron-voice.md
- sessions/2026-03-30-kos-bootstrap-cron-jobs-1password-gemini.md
- sessions/2026-03-31-openclaw-update-oauth-models-monitor.md
- sessions/2026-03-31-subscriptions-expense-tracking-3.md
- sessions/2026-03-30-kai-new-capabilities-reminders-audio-2.md
- sessions/2026-03-30-kai-reminders-audio-implementation-3.md
- sessions/2026-03-30-kai-fixes-kos-openclaw-monitor-4.md
- sessions/2026-03-31-kai-cron-briefings-waste-calendar-memory-2.md
last_updated: '2026-04-01'
version: 4
---

# Cron Jobs, Sub-Agents, and Automation

OpenClaw supports scheduled tasks (cron jobs), on-demand sub-agents, document handling, and voice messaging. This guide covers practical automation patterns.

## Cron Jobs

### Basic Syntax

```bash
openclaw cron add \
  --name "Job Name" \
  --cron "0 7 * * 1" \
  --tz "America/New_York" \
  --session isolated \
  --message "Your instructions to the agent" \
  --model "opus" \
  --announce \
  --channel whatsapp
```

| Parameter | Purpose |
|-----------|---------|
| `--cron` | Standard cron expression (minute hour day month weekday) |
| `--tz` | Timezone for scheduling |
| `--session isolated` | Run in a fresh session (no conversation history bleed) |
| `--model` | Model to use (opus for complex analysis, sonnet for routine tasks) |
| `--announce` | Send the result to the configured channel |
| `--channel` | Which channel receives the announcement |

### Cost-aware model selection

For recurring jobs, prefer subscription-backed models and keep per-token API usage out of the normal path:

- **OpenAI**: use `openai-codex/*` via OAuth when available
- **Google**: use `google-gemini-cli/*` via OAuth when available
- **Anthropic**: use `claude-cli/*` via Max subscription
- **Fallback keys**: keep `OPENAI_API_KEY`, `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, and `OPENROUTER_API_KEY` only as emergency fallbacks

For daily/weekly monitoring jobs, make the prompt explicitly say what to do when there is no change so the agent can SKIP instead of fabricating output.

Use the same pattern for family briefings, waste reminders, and release monitors: define the data sources, define the fallback behavior, and keep the output short enough to read in chat.

### Example Schedules

**Daily routines**:
- Morning briefing (7:00 AM): Urgent emails, today's calendar, weather
- Midday check (12:00 PM): Medication reminders, appointments, shopping list
- Evening recap (9:00 PM): Day summary, open tasks, tomorrow's agenda

**Weekly routines**:
- Monday review: Weekly summary, planning, budget check
- Sunday meal planning: Weekly menu + grocery list
- Sunday night backup: Git commit of workspace

**Monthly routines**:
- First of month: Bill payment reminders
- Birthday and anniversary alerts (3 days advance notice)
- Subscription renewal warnings (7 days advance)

**Periodic watchers**:
- Urgent email monitor (every 20 minutes)
- Calendar check for next 2 hours (every hour)

## Sub-Agents

Sub-agents are spawned on demand for specific tasks. The **cron + sub-agent** pattern is particularly powerful: a cron job triggers a sub-agent with a high-capability model for resource-intensive analysis.

### Common Sub-Agent Types

| Agent | Purpose |
|-------|---------|
| Research agent | Deep research on a topic |
| Email triage agent | Categorize and prioritize emails |
| Budget agent | Analyze spending patterns |
| Shopping agent | Compare prices, find deals |
| Schedule optimizer | Resolve calendar conflicts |
| Document analyzer | Extract data from PDFs and images |
| Travel planner | Coordinate trip logistics |

## Document and Media Handling

| Capability | Support | Notes |
|-----------|---------|-------|
| Archive documents to Drive | Native | Via GWS Drive skills |
| Read/analyze PDFs | Native | The model reads PDFs directly |
| Edit PDFs | Requires setup | Needs `pdftk` in Docker sandbox |
| Image → PDF conversion | Requires setup | Needs `img2pdf` |
| Image editing | Requires setup | Needs ImageMagick |
| Git backup | Native | Via exec + cron |

## Voice Messaging (TTS/STT)

### Receiving Voice Messages (STT)

Transcription works automatically using OpenAI Whisper — no configuration needed on the user side.

### Sending Voice Messages (TTS)

Three provider options:

| Provider | Quality | Cost | Italian Support | Customization |
|----------|---------|------|-----------------|---------------|
| **Edge TTS** | Good | **Free** | Yes (ElsaNeural, DiegoNeural) | 400+ voices |
| **OpenAI TTS** | Good | Paid | Yes | 6 preset voices |
| **ElevenLabs** | Excellent | Paid | Yes (multilingual_v2) | 1000+ voices, cloning |

**Recommendation for getting started**: Edge TTS is free and provides solid quality for multiple languages.

### Configuration Example

```json
{
  "messages": {
    "tts": {
      "auto": "always",
      "provider": "elevenlabs",
      "elevenlabs": {
        "voiceId": "VOICE_ID",
        "modelId": "eleven_multilingual_v2",
        "languageCode": "en",
        "voiceSettings": {
          "stability": 0.6,
          "similarityBoost": 0.75,
          "speed": 1.0
        }
      }
    }
  }
}
```

### Voice Cloning

ElevenLabs supports instant voice cloning with 1–5 minutes of clean audio.

> ⚠️ **Privacy consideration**: Voice is classified as biometric data under GDPR. Cloning a real person's voice without their explicit consent raises legal and ethical concerns.

## Fetching Promotions and Local Events

### Supermarket Deals

**Recommended approach**: Use aggregator sites (e.g., PromoQui, VolantinoFacile) by postal code rather than scraping individual retailer websites. Aggregators provide all chains in one request.

**Limitations**:
- National chains: Direct fetch is reliable
- Regional chains: Depends on whether they have a web-accessible flyer
- App-only flyers: Not reachable via web scraping
- CAPTCHA/anti-scraping: Browser tool helps but doesn't guarantee access

### Local Events

Use a multi-source pipeline:
1. Fresh web search results
2. Eventbrite (local filter)
3. Municipal website events section
4. Local newspaper RSS feeds

### Social Media

Technically possible with persistent browser profiles (`browser create-profile`), but Meta detects automation and may ban accounts. **Use for reading only, never for posting.**

## What's Next

- [Agent Fleet Design](../concepts/agent-fleet-design.md) — Designing sub-agent hierarchies
- [Skills and Slash Commands](skills-and-slash-commands.md) — Extend your automation capabilities
