---
title: "Deep Research — GLM-5.1 (Z.AI) as primary LLM, subscription billing verified"
date: "2026-04-08"
author: "kos-domus"
status: "ready"
tags: ["mcp", "mcp-servers", "automation", "configuration", "api", "performance"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Integrate GLM-5.1 (by Z.AI) as the primary LLM for the deep research pipeline, using Kos's Z.AI Coding Pro yearly subscription to avoid pay-per-token costs.

## Context
- Pipeline was using Gemini CLI (primary, Google subscription) → OpenAI API (fallback, pay-per-token)
- Kos has a Z.AI Coding Pro yearly plan with `ZAI_API_KEY` configured
- OpenAI API confirmed as separate billing from ChatGPT Pro subscription — every API call costs extra
- Goal: use subscription-based LLMs for all extraction and planning, zero extra costs

## Steps Taken

### 1. Investigated Z.AI API and billing
Discovered Z.AI has two API endpoints:
- **Regular endpoint** (`api.z.ai/api/paas/v4`) — pay-per-token, requires credit balance
- **Coding endpoint** (`api.z.ai/api/coding/paas/v4`) — included in Coding Pro subscription

Tested both with `ZAI_API_KEY`:
- Regular endpoint: `glm-5.1` returned "Insufficient balance or no resource package" (no pay-per-token credits)
- Coding endpoint: `glm-5.1` returned a valid response — subscription covers this

**Result**: Coding endpoint confirmed as the billing path for the Pro subscription.

### 2. Added GLM-5.1 as primary extractor
Updated `llm-extract.ts` with new `callZAI()` function:
- Endpoint: `https://api.z.ai/api/coding/paas/v4/chat/completions`
- Model: `glm-5.1`
- Handles reasoning model quirk: GLM-5.1 may return `reasoning_content` separately from `content`
- Strips markdown fences from output
- Initial timeout 45s was too short for reasoning model — increased to 90s

**Result**: GLM-5.1 extraction working. Extracted Aave at $95.02, category "lending" from Decrypt.co.

### 3. Added GLM-5.1 for auto-research planning
Updated `auto-research.ts` with same `callZAI()` pattern.

Tested: "Best nocode tools for building SaaS in 2026" → GLM-5.1 planned 8 sites (Bubble, G2, Product Hunt, Reddit r/nocode, nocode.tech, Zapier blog, Softr, Capterra) with 13-field schema.

**Result**: Planning works on subscription billing.

### 4. Updated MCP config
Added `ZAI_API_KEY` to `.mcp.json` env for the deep-research MCP server.

**Result**: MCP server will have access to Z.AI on next session restart.

## Configuration Changes
- `~/job-desk/tools/deep-research/src/normalizer/llm-extract.ts` — added `callZAI()`, GLM-5.1 as priority 1, timeout 90s
- `~/job-desk/tools/deep-research/src/auto-research.ts` — added `callZAI()`, GLM-5.1 as priority 1 for planning, timeout 90s
- `~/job-desk/.mcp.json` — added `ZAI_API_KEY` env var for deep-research MCP server

## Key Discoveries
- **Z.AI has two billing paths**: `api.z.ai/api/paas/v4` (pay-per-token) vs `api.z.ai/api/coding/paas/v4` (subscription). The Coding Pro plan only covers the coding endpoint. Using the wrong endpoint returns "Insufficient balance."
- **GLM-5.1 is a reasoning model**: It returns `reasoning_content` separately from `content`. With `max_tokens: 50`, all tokens went to reasoning and content was empty. Need `max_tokens: 4000` for extraction tasks.
- **GLM-5.1 needs 90s timeout**: As a reasoning model, it takes 30-40s for extraction (vs 3s for GPT-5.4-mini). The initial 45s timeout was too tight.
- **GLM-5.1 extraction is focused**: It extracted exactly what the schema asked for (3 precise fields) vs Gemini which dumped 242 entries. Better for structured extraction where you want specific data, not a data dump.
- **OpenAI API is completely separate from ChatGPT Pro**: Confirmed via [OpenAI docs](https://help.openai.com/en/articles/9039756-billing-settings-in-chatgpt-vs-platform). The $200/month Pro subscription covers chatgpt.com + Codex CLI. The API (`api.openai.com`) bills separately per token.
- **Three subscription-based LLMs available**: Z.AI Coding Pro (GLM-5.1), Google subscription (Gemini CLI), ChatGPT Pro (only via OpenClaw `openai-codex` provider, not raw API). First two can be used for extraction; third cannot without full agent context overhead.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| Z.AI "Insufficient balance or no resource package" | Using regular endpoint instead of coding endpoint | Changed base URL to `api.z.ai/api/coding/paas/v4` |
| GLM-5.1 timeout at 45s | Reasoning model needs more time to think + generate | Increased timeout to 90s |
| GLM-5.1 empty content with `max_tokens: 50` | All tokens consumed by reasoning, none left for content | Set `max_tokens: 4000` |

## Final State
**LLM priority stack (all on subscription billing):**

| Priority | Provider | Model | Billing | Speed |
|----------|----------|-------|---------|-------|
| 1st | Z.AI coding endpoint | GLM-5.1 | Coding Pro yearly (free) | ~35s |
| 2nd | Gemini CLI | gemini-2.5-flash | Google subscription (free) | ~15s |
| 3rd | OpenAI API | gpt-5.4-mini | Pay-per-token (last resort) | ~3s |

Normal operation: zero extra LLM costs. OpenAI API only triggers if both Z.AI and Gemini fail.

## Open Questions
- GLM-5.1 speed: 35s per extraction is slower than Gemini (15s) or OpenAI (3s). For multi-site jobs this adds up. Consider using GLM-4.7-Flash (free tier, faster) for extraction and reserving GLM-5.1 for planning where quality matters more.
- Z.AI Coding Pro rate limits: not yet documented. Monitor for 429s under heavy use.
- The coding endpoint supports `glm-5.1` but may also support other models (GLM-5-Turbo, GLM-4.7). Test which models are included in the Pro plan.
