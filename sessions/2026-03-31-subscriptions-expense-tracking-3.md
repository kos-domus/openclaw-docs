---
title: "Subscription expense tracking system: all AI/tech costs centralized for Kos"
date: "2026-03-31"
author: "kos-domus"
status: "ready"
tags: ["configuration", "multi-agent", "performance", "automation"]
openclaw_version: "2026.3.28"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote (Mac → Ubuntu SSH)"
  model: "claude-opus-4-6"
---

## Objective
Set up a centralized expense tracking system for all AI/tech subscriptions and API costs, maintained by Kos. All values normalized to EUR.

## Context
- System uses 5 paid services: Anthropic, OpenAI, Google, 1Password, Firecrawl
- After migrating to OAuth subscriptions (OpenAI Codex, Google Gemini CLI), per-token API costs should be ~€0
- Kos already had an API monitoring task but it lacked subscription cost awareness
- User provided actual billing screenshots for all 5 providers

## Steps Taken

### 1. Created master subscriptions registry
Created `~/.openclaw/workspace-cos/reports/subscriptions-and-costs.json` containing:
- All 5 subscriptions with real billing data (amounts, dates, plans, histories)
- 6 fallback API keys reference (loaded via 1Password)
- All 8 cron job IDs for token usage tracking
- Dashboard URLs for each provider
- Monthly and annual cost totals

**Result**: Single source of truth for all AI/tech costs.

### 2. Recorded Anthropic billing data
From user's invoice screenshots:
- Plan: Max, billing day: 2nd of each month
- Current: €109.80/month
- History: Pro (€21.96) → Max upgrade (€96.84, Feb 2026) → Max (€109.80, Mar 2026)
- Next invoice: April 2, 2026

**Result**: Full billing history recorded.

### 3. Recorded OpenAI billing data
From user's invoice screenshot:
- Plan: ChatGPT Plus, billing day: 18th of each month
- Amount: $24.40/month ($20.00 + $4.40 tax) → €21.93/month
- Consistent since January 2026
- Next invoice: April 18, 2026

**Result**: Billing history and tax breakdown recorded.

### 4. Recorded Google billing data
From user's order history:
- Plan: Google One AI Premium 5TB, annual billing
- Amount: €249.99/year (€20.83/month equivalent)
- Paid: March 31, 2026. Expires: March 31, 2027
- Upgraded from 100GB (€19.99/year) — previous plan refunded

**Result**: Annual billing recorded with upgrade history.

### 5. Recorded 1Password billing data
From user's upcoming invoice:
- Plan: Families (Annual), billing date: April 11
- Amount: $65.77/year ($71.88 - 25% discount + 22% tax) → €59.10/year (€4.93/month)
- First payment: April 11, 2026
- 25% discount valid until March 28, 2027

**Result**: Annual billing with discount details recorded.

### 6. Recorded Firecrawl billing data
From user's receipt:
- Plan: Hobby (Credits), annual billing
- Amount: $190.00/year (charged €170.78 at 1 USD = 0.8988 EUR)
- Period: March 26, 2026 — March 26, 2027

**Result**: Annual billing with EUR conversion recorded.

### 7. Normalized all costs to EUR
Converted all USD amounts to EUR using the exchange rate from the Firecrawl receipt (1 USD = 0.8988 EUR). All monthly totals now in EUR.

**Result**: Unified cost view in EUR.

### 8. Updated Kos's bootstrap files
- Added "Subscriptions & Costi AI" section to Kos's MEMORY.md with subscription table, API key fallback references, dashboard URLs, and tracking rules
- Updated `tasks/api-usage-cost-monitoring.md` to reference the master subscriptions file
- Rule: if API key fallback usage detected → investigate why subscription provider failed

**Result**: Kos has full awareness of all costs and where to find the data.

## Configuration Changes
- `~/.openclaw/workspace-cos/reports/subscriptions-and-costs.json` — NEW: master registry of all subscriptions with real billing data
- `~/.openclaw/workspace-cos/MEMORY.md` — Added "Subscriptions & Costi AI" section
- `~/.openclaw/workspace-cos/tasks/api-usage-cost-monitoring.md` — Added reference to subscriptions file

## Key Discoveries
- **Anthropic Max is the biggest cost at €109.80/month** — 64% of the total monthly spend. Worth monitoring whether the Max tier is fully utilized vs Pro.
- **Annual plans save significantly**: Google (€20.83/mo vs monthly rate), 1Password (25% discount), Firecrawl ($190/year vs monthly credits)
- **All per-token API costs should be €0**: With OpenAI Codex OAuth, Google Gemini CLI OAuth, and Claude CLI, agents use subscription quotas. The 6 API keys in 1Password are strictly fallback.
- **Mixed currency billing complicates tracking**: Anthropic and Google bill in EUR, OpenAI and 1Password in USD. Kos needs a reference exchange rate (currently 1 USD = 0.8988 EUR from Firecrawl receipt).

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| N/A — configuration session | — | — |

## Final State
- **5 subscriptions tracked** with real billing data, history, and EUR normalization
- **Total monthly cost: €171.72** (~€2,060/year)
- **Billing calendar**:
  - 2nd monthly: Anthropic Max €109.80
  - 18th monthly: OpenAI Plus €21.93
  - Mar 26 annual: Firecrawl €170.78 (€14.23/mo)
  - Mar 31 annual: Google AI Premium €249.99 (€20.83/mo)
  - Apr 11 annual: 1Password Families €59.10 (€4.93/mo)
- **Variable API costs: ~€0** (subscriptions cover all usage)
- Kos's API monitoring report will now include subscription awareness

## Open Questions
- Should Kos generate a monthly subscription cost summary alongside the API monitoring report?
- How to track exchange rate fluctuations for USD-billed services (OpenAI, 1Password)?
- Is the Anthropic Max tier fully utilized, or would Pro ($20/mo) suffice for current usage?
- Should Firecrawl credit usage be monitored to avoid running out before renewal?
