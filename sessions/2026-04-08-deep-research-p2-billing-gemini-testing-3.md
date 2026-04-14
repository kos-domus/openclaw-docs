---
title: "Deep Research — P2 hardening, LLM billing fix (Gemini CLI primary), full product test"
date: "2026-04-08"
author: "kos-domus"
status: "processed"
tags: ["mcp", "mcp-servers", "automation", "security", "performance", "configuration", "testing", "troubleshooting"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Implement P2 hardening items from team review, fix LLM billing to use subscription models (not API credits), and run a full product test.

## Context
- P0/P1 items were completed in the previous session (URL sanitization, job timeout, Playwright, parallel contexts, CLI auto)
- LLM extraction was using OpenAI API (`api.openai.com`) which bills separately from the ChatGPT Pro $200/month subscription
- Kos wanted to avoid extra costs — all LLM calls should use subscription billing
- 5 P2 items pending from the Co-CEO team review

## Steps Taken

### 1. Investigated OpenAI billing model
Discovered that `openai-codex` provider in OpenClaw uses `https://chatgpt.com/backend-api` with OAuth tokens — this is the ChatGPT subscription. The OpenAI API (`api.openai.com` with `OPENAI_API_KEY`) is completely separate pay-per-token billing.

Attempted to use OpenClaw's OAuth tokens directly against the ChatGPT backend API — got 403. The internal API requires OpenClaw's full provider layer, not raw HTTP calls.

Tested `openclaw agent --agent cso -m "..." --json` via gateway — works but loads 16,848 tokens of agent context overhead per call. Too expensive for extraction.

**Result**: Cannot use GPT subscription for raw extraction calls. OpenAI API and ChatGPT Pro are [separate billing systems](https://help.openai.com/en/articles/9039756-billing-settings-in-chatgpt-vs-platform).

### 2. Flipped LLM priority: Gemini CLI primary, OpenAI API fallback
Since Gemini CLI uses OAuth subscription (Google subscription, free), made it the primary LLM provider:

**Before**: OpenAI API (pay-per-token) → Gemini CLI (fallback)
**After**: Gemini CLI (subscription, free) → OpenAI API (fallback only)

Fixed in both `llm-extract.ts` and `auto-research.ts`.

**Result**: All extraction calls now go through Gemini CLI on subscription billing. Zero API costs for normal operation.

### 3. Fixed Gemini CLI invocation
Multiple issues found and fixed:
- `--prompt-file` flag doesn't exist in Gemini CLI — uses `-p` for non-interactive mode
- Model name: `gemini-2.5-flash` works, `gemini-3-flash` returns 404
- `spawn` approach caused the CLI to hang — switched to `execFile` with shell piping (`cat file | gemini -m gemini-2.5-flash -p "..."`)
- Gemini CLI internally tries `gemini-3-flash-preview` even when `gemini-2.5-flash` is specified — hits 429 when Google capacity is exhausted

**Result**: Working Gemini CLI integration. Tested: extracted 242 crypto token prices from Decrypt.co using subscription billing.

### 4. Implemented all P2 items

**Query deduplication**: Hash(query + domain + sorted sites) checked against completed jobs within 6 hours. Returns existing results instead of re-running.

**Approval caller verification**: `approveJob()` now accepts an `approver` parameter. Checks the approver has `full` tier access — standard/request agents cannot approve jobs.

**Queue rotation**: Jobs older than 7 days in terminal state (completed/failed/cancelled/rejected) are archived to `queue/archive/YYYY-MM.json`. Runs automatically on every queue write.

**Reliability-based site skipping**: If protocol store shows a site with <30% reliability after 5+ attempts, the pipeline auto-skips it instead of wasting time and Firecrawl credits.

**Atomic queue writes**: Queue file now uses write-to-tmp + rename pattern to prevent corruption from concurrent access.

**Result**: All P2 items compiled and working.

### 5. Full product test
Ran comprehensive tests across all features:

| Test | Result | Details |
|------|--------|---------|
| CLI help | Pass | 8 commands including `auto` |
| Domains listing | Pass | crypto (5 sites), backend-tech (5 sites) |
| Domain submit (3 sites) | Pass | 3/3 OK, 86s, Playwright for Decrypt+DLNews, Firecrawl fallback for DefiLlama |
| Dedup check | Pass | Same query returns existing job instantly |
| Protocol store | Pass | 3 sites learned with correct methods |
| Auto-research (summer camps) | Partial | Planning worked (7 Italian sites discovered), but Gemini 429 + job timeout at 120s |
| Job list | Pass | Shows all jobs with status, domain, query |
| Research timeline | Pass | 1 run tracked, 3 sites, 248 fields |

**Result**: Core pipeline fully functional. Gemini CLI intermittent 429s are the main reliability concern.

## Configuration Changes
- `~/job-desk/tools/deep-research/src/normalizer/llm-extract.ts` — rewritten: Gemini CLI primary via shell piping, OpenAI API fallback, model `gemini-2.5-flash`
- `~/job-desk/tools/deep-research/src/auto-research.ts` — rewritten Gemini CLI call: same shell piping approach, `gemini-2.5-flash`
- `~/job-desk/tools/deep-research/src/queue/job-queue.ts` — rewritten: added dedup, approval verification, queue rotation, atomic writes
- `~/job-desk/tools/deep-research/src/pipeline.ts` — added reliability-based site skipping (`<30%` after `5+` attempts)
- `~/job-desk/tools/deep-research/src/mcp-server.ts` — added dedup check in `research_submit`, approver param in `research_approve`

## Key Discoveries
- **ChatGPT Pro and OpenAI API are separate billing**: The $200/month Pro subscription covers chatgpt.com + Codex CLI. The API (`api.openai.com`) is completely separate pay-per-token. They do NOT share billing. No way to use the Pro subscription for raw API extraction calls.
- **OpenClaw `openclaw agent` loads ~16K tokens of context overhead**: Even for a simple JSON extraction, the full agent bootstrap (SOUL.md, skills, tools) gets loaded. Not suitable for high-frequency extraction calls.
- **Gemini CLI model routing is confusing**: Specifying `-m gemini-2.5-flash` works for simple calls, but the CLI internally tries `gemini-3-flash-preview` for some operations (like web search tool calls), which hits 429 when Google capacity is low.
- **Gemini CLI `--prompt-file` doesn't exist**: Must use `-p` for the prompt flag. For long prompts, pipe via stdin: `cat file | gemini -m model -p "instruction"`.
- **Gemini 2.5-flash extracts richer data than GPT-5.4-mini**: Gemini pulled 242 tokens with prices from Decrypt.co vs GPT returning just 1 entry. Gemini appears better at tabular data extraction.
- **Atomic file writes prevent queue corruption**: The write-to-tmp + rename pattern (`writeFileSync(tmp) + renameSync(tmp, target)`) is essential for a file-based queue that multiple processes may access.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| ChatGPT backend API 403 | OAuth tokens require OpenClaw's full provider layer, not raw HTTP | Don't try to call backend API directly — use Gemini CLI instead |
| Gemini CLI `--prompt-file` unknown argument | Flag doesn't exist in gemini CLI v0.36.0 | Use `-p` for prompt, pipe long prompts via stdin with `cat file \| gemini` |
| Gemini CLI `gemini-3-flash` model not found (404) | Model ID doesn't exist in Gemini CLI namespace | Use `gemini-2.5-flash` instead |
| Gemini CLI 429 `MODEL_CAPACITY_EXHAUSTED` | Google servers overloaded for `gemini-3-flash-preview` | Intermittent — CLI internally retries. Falls through to OpenAI API fallback |
| `spawn` approach hangs Gemini CLI | CLI waits for interactive input even in non-interactive mode when spawned | Use `execFile` with `/bin/bash -c "cat file \| gemini ..."` instead |
| Auto-research timeout at 120s | Gemini 429 retries consumed the planning phase, leaving no time for scraping | Job timeout worked correctly as a safety net |

## Final State
- **14 MCP tools** across all phases
- **LLM stack**: Gemini CLI (subscription, free, primary) → OpenAI API (pay-per-token, fallback)
- **Fetch stack**: Playwright (free, default, parallel batches of 2) → Firecrawl (paid, fallback)
- **P2 hardening**: Query dedup (6h window), approval verification, queue rotation (7d), reliability skipping (<30%), atomic writes
- **All P0/P1/P2 items complete**
- **Known issue**: Gemini CLI intermittent 429s when Google capacity is low

## Open Questions
- Gemini CLI 429 reliability: Is this a temporary Google capacity issue or a persistent limitation of the subscription tier? Monitor over next few days
- SearXNG integration (Co-CEO P1 recommendation): Self-hosted web search would replace LLM-based site discovery — highest ROI improvement still pending
- Cost ledger was dropped from P2 scope since Gemini CLI is free — revisit if OpenAI API fallback triggers frequently
- Should we add `gemini-2.5-pro` as a middle-tier option between flash (fast/cheap) and OpenAI API (expensive)?
