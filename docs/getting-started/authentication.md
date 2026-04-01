---
title: 'Authentication: API Keys vs Setup Tokens'
slug: authentication
category: getting-started
tags:
- configuration
- api
- authentication
- troubleshooting
sources:
- sessions/2026-03-14-anthropic-auth-apikey-vs-setuptoken.md
last_updated: '2026-03-30'
version: 2
---

# Authentication: API Keys vs Setup Tokens

OpenClaw supports two authentication methods for Anthropic. Choosing the wrong one is the #1 cause of unexpected rate limits.

## The Two Token Types

| | API Key | Setup Token |
|---|---------|-------------|
| Format | `sk-ant-api03-...` | `sk-ant-oat01-...` |
| Source | console.anthropic.com | `claude setup-token` CLI |
| Billing | Pay-per-token (usage-based) | Included in Pro/Max subscription |
| Rate limits | Low (depends on spend tier) | High (subscription-tier limits) |
| Best for | Production APIs, programmatic access | Personal agents, bots, always-on use |

## The Problem: Rate Limits Hit Despite Having Credits

API key rate limits are determined by your **spend tier**, not your credit balance. You can have $100 in API credits and still hit rate limits after a few dozen requests if you're on a low tier.

If you have an Anthropic Max subscription ($90/month), switching to a setup token gives you dramatically higher rate limits at no additional cost.

## Generating a Setup Token

```bash
# Install Claude Code CLI (if not already installed)
npm install -g @anthropic-ai/claude-code

# Generate the setup token
claude setup-token
```

This opens a browser for authentication. After signing in, it generates a token starting with `sk-ant-oat01-...`.

**Key insight**: The token can be generated on **any machine** (e.g., your Mac with a browser) and then transferred to the headless server. The token is portable.

## Configuring OpenClaw with Setup Token

### Option 1: Re-run onboarding

```bash
openclaw onboard
# Select "Anthropic token (setup-token)" — NOT "Anthropic API key"
# Paste the sk-ant-oat01-... token
```

### Option 2: Edit config directly

If the bot is unreachable due to rate limits (every API call fails), edit the configuration file directly:

```bash
nano ~/.openclaw/openclaw.json
```

Or use the web dashboard at `http://localhost:18789` if it's still accessible.

### Option 3: Abort and restart onboarding

If onboarding gets stuck, press `Ctrl+C` to abort. Running `openclaw onboard` again starts fresh.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `API rate limit reached` with credits available | API key rate limits are tier-based, not balance-based | Switch to setup token |
| Bot completely unreachable after rate limit | All API calls fail in a cascade | Edit `openclaw.json` directly or use dashboard |
| `sk-ant-oat01` requested but you have `sk-ant-api03` | Wrong auth method selected in onboarding | Re-run onboarding, select "setup-token" |

## When to Use Each

- **Setup token**: Personal bots, always-on agents, development — anything tied to your subscription
- **API key**: Production deployments billed separately, multi-tenant services, CI/CD pipelines

## Open Questions

- Setup token refresh: Token expiration behavior and automatic renewal are not fully documented. Monitor for auth failures and regenerate if needed.
- Usage monitoring: There is currently no way to monitor Max subscription usage separately for OpenClaw vs. other Claude tools.
