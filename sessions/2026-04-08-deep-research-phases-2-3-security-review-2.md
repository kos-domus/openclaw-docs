---
title: "Deep Research Pipeline — Phases 2-3, full team security review, P0/P1 hardening"
date: "2026-04-08"
author: "kos-domus"
status: "ready"
tags: ["mcp", "mcp-servers", "automation", "security", "multi-agent", "performance", "configuration", "cli"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Complete Phases 2 and 3 of the deep research pipeline, run a full team review (Co-CEO consulting CSO, Backend Expert, Frontend Specialist, Orchestration Architect), and implement all P0/P1 fixes.

## Context
- Phase 1 was completed earlier in the same day (separate session log)
- Pipeline had: MCP server (7 tools), Firecrawl-only fetching, keyword-based extraction, file-based queue
- Needed: protocol store, scheduling, LLM extraction, Playwright, agent-initiated research, research memory, security hardening

## Steps Taken

### 1. Phase 2: Protocol Store
Built a protocol store at `protocols/sites.json` that learns from every scrape:
- Records per-hostname: method, avg fetch time, success/fail count, reliability score
- Discovers extraction patterns (regex) from successful extractions
- Discovers content anchors (headings near extracted data)
- Normalizer checks stored patterns first (free, instant) before falling back to LLM

Cleared stale Phase 1 patterns after discovering they contained garbage keyword-matched values (e.g., `tvl: "Fees & Revenue"`). Fixed merge order: LLM results take priority over protocol store.

**Result**: Protocol store operational, re-learning from LLM-quality extractions.

### 2. Phase 2: Scheduling Scaffolding
Built scheduling system (types, CRUD, MCP tools) but all schedules start disabled by default. Ready to wire to OpenClaw cron when needed.

New types: `ScheduleConfig` with id, domain, project, query, sites, cron expression, enabled flag.
New MCP tools: `research_schedules`, `research_schedule_create`.

**Result**: Scheduling defined but inactive — no unnecessary cost or compute.

### 3. Phase 3: LLM-Assisted Extraction
Replaced Phase 1 keyword matching with LLM-powered structured extraction:

**Provider stack:**
- Primary: gpt-5.4-mini (OpenAI API, $200 subscription)
- Fallback: gemini-3-flash via `gemini` CLI (OAuth subscription billing, not API credits)

**Extraction priority:**
1. Protocol store patterns (free, instant)
2. LLM extraction (gpt-5.4-mini → gemini-3-flash)
3. Generic keyword matching (legacy fallback)

**Key fixes during implementation:**
- OpenAI gpt-5.4-mini requires `max_completion_tokens` not `max_tokens` (API 400 error)
- Gemini CLI at `/home/kos/.npm-global/bin/gemini`, needed explicit path
- `.bashrc` updated: `op-env-cached.sh` instead of `op-env.sh` for reliable MCP env var loading

**Quality comparison (DefiLlama):**
- Before (keyword): `tvl: "Fees & Revenue"` (garbage)
- After (LLM): `tvl: 24194000000, price: 87.18, market_cap: 1325000000` (real data)

**Result**: 4/4 sites on multi-site test, 14s total, real structured data.

### 4. Phase 3: Playwright Browser Support
Installed Playwright + Chromium for free local rendering (vs Firecrawl €14/month):

```bash
npm install playwright
npx playwright install chromium  # 170MB Chromium + 112MB headless shell
```

**Strategy flipped: Playwright-first (free), Firecrawl fallback:**
1. New site → Playwright tries (free)
2. If content < 200 chars → auto-fallback to Firecrawl (paid)
3. Protocol store remembers which method works → skips failed attempts next time

Browser launch args optimized for mini PC: `--disable-gpu`, `--disable-dev-shm-usage`, `--no-sandbox`, resource-blocking routes for images/analytics.

Built `nodeToMarkdown` converter in Playwright's `page.evaluate()` — converts DOM to markdown including table support.

**Result**: Decrypt.co served by Playwright (free, 3.2s), DefiLlama auto-fell-back to Firecrawl (heavy JS dashboard).

### 5. Phase 3: Agent-Initiated Research
Built `research_auto` — free-form query to full structured results in one call:

1. LLM (gpt-5.4-mini) plans: discovers 5-8 authoritative sites + defines extraction schema
2. Pipeline scrapes all sites (Playwright/Firecrawl)
3. LLM extracts structured data per site
4. Optional: save domain config as YAML for reuse (`saveDomain: true`)

**Test**: "Best PostgreSQL alternatives for time-series data" → auto-discovered Timescale, InfluxDB, QuestDB, ClickHouse, MongoDB, AWS Timestream, Azure Data Explorer → 7/7 sites, 12-14 fields per site, 53 seconds.

**Result**: MCP tool `research_auto` + CLI command `research auto` both working.

### 6. Phase 3: Research Memory
Built diffing and timeline capabilities:
- `findPreviousResult()`: finds most recent previous result for same project+domain
- `diffResults()`: compares current vs previous — new sites, removed sites, field value changes
- `getResearchTimeline()`: chronological history of all runs

New MCP tools: `research_diff`, `research_timeline`.

**Result**: Agents can track how research data changes over time.

### 7. Full Team Review via Co-CEO
Co-CEO consulted all four specialist agents. Key findings per domain:

**CSO:**
- P0: No URL validation — SSRF risk (private IPs, file:// URIs)
- P0: No job timeout — runaway compute risk
- Approval endpoint has no caller verification
- Job queue has no file locking

**Backend Expert:**
- Sequential Playwright is the bottleneck — parallelize to 2-3 contexts
- LLM context truncated at 6,000 chars — should be 12,000
- No cost tracking (tokens, Firecrawl calls)
- Queue grows unbounded

**Frontend Specialist:**
- `auto` command missing from CLI (MCP-only)
- Response shapes inconsistent across MCP tools

**Orchestration Architect:**
- No job-level timeout
- No query deduplication
- No reliability-based site skipping

**On web search engines:** YES, add SearXNG (self-hosted, free, ~200MB RAM). Biggest quality improvement — replaces "LLM imagining URLs" with "LLM choosing from real search results."

**On browser diversity (Firefox/WebKit):** DEFER. No evidence Chromium is being blocked. If needed: add `playwright-extra` stealth plugin first (free, 10 lines).

### 8. P0/P1 Implementation
Implemented all priority items:

**P0: URL Sanitization (`url-guard.ts`)**
- Blocks: private IPs (RFC 1918, loopback, link-local), file:/ftp:/javascript: schemes, metadata endpoints, hostnames resolving to private IPs via DNS
- Validates every URL before Playwright or Firecrawl fetch
- Tested: blocks `192.168.1.1`, `localhost`, `file:///etc/passwd`, `169.254.169.254/metadata`; passes `defillama.com`, `decrypt.co`

**P0: Job Timeout**
- `Promise.race` with configurable timeout (default 120s via `RESEARCH_JOB_TIMEOUT` env var)
- On timeout: closes browser, marks job failed, returns empty result

**P1: CLI `auto` Command**
- `research auto --query "..."` or `research auto your question here`
- Tested: "Best open source vector databases for RAG" → 7/7 sites (Milvus, Weaviate, Chroma, Qdrant, OpenSearch, Lucene, pgvector), 34s, 12-14 fields each

**P1: Parallel Playwright**
- Changed from sequential to batches of 2 concurrent pages in same browser instance
- Cuts job duration by ~50% with minimal extra RAM (~50MB per context)

**P1: LLM Context Increase**
- Changed from 6,000 to 12,000 chars in `llm-extract.ts`
- Better extraction for data-rich pages

## Configuration Changes
- `~/job-desk/tools/deep-research/src/protocol-store.ts` — new: protocol store module
- `~/job-desk/tools/deep-research/src/schedules.ts` — new: scheduling scaffolding
- `~/job-desk/tools/deep-research/src/normalizer/llm-extract.ts` — new: LLM extraction (OpenAI + Gemini CLI)
- `~/job-desk/tools/deep-research/src/fetchers/playwright.ts` — new: Playwright fetcher
- `~/job-desk/tools/deep-research/src/auto-research.ts` — new: agent-initiated research planner
- `~/job-desk/tools/deep-research/src/research-memory.ts` — new: diff and timeline
- `~/job-desk/tools/deep-research/src/url-guard.ts` — new: SSRF protection
- `~/job-desk/tools/deep-research/src/pipeline.ts` — rewritten: Playwright-first strategy, URL validation, job timeout, parallel batching
- `~/job-desk/tools/deep-research/src/normalizer/normalize.ts` — rewritten: async, LLM-first with protocol store + keyword fallback
- `~/job-desk/tools/deep-research/src/mcp-server.ts` — expanded: 14 tools total
- `~/job-desk/tools/deep-research/src/cli.ts` — added: `auto` command
- `~/job-desk/tools/deep-research/src/types.ts` — added: SiteProtocol, ScheduleConfig types
- `~/job-desk/.mcp.json` — added: `OPENAI_API_KEY` env var for deep-research MCP
- `~/.bashrc` — changed: source `op-env-cached.sh` with fallback to `op-env.sh`
- `~/job-desk/tools/deep-research/package.json` — added: `playwright` dependency
- Memory files updated: `project_etl_web_research.md`, 3 new reference memories

## Key Discoveries
- **gpt-5.4-mini uses `max_completion_tokens` not `max_tokens`**: The older parameter name causes a 400 error. All GPT-5.x models require the new parameter.
- **Gemini CLI is at `~/.npm-global/bin/gemini`**: Not in default PATH for child processes. Must pass explicit PATH or use absolute path.
- **`.bashrc` non-interactive guard**: The `case $- in *i*) ;; *) return;; esac` block means env vars from `.bashrc` don't load for MCP server processes (spawned non-interactively). The `.mcp.json` env config must reference vars that exist in the parent process.
- **Protocol store merge order matters**: `{ ...llmResult, ...protocolStore }` means protocol store overrides LLM. Must be `{ ...protocolStore, ...llmResult }` so LLM (higher quality) wins.
- **Playwright Chromium can't render all JS dashboards**: DefiLlama returns <200 chars of content via Playwright. The auto-fallback to Firecrawl handles this gracefully.
- **Parallel Playwright contexts share one browser**: Running 2-3 pages concurrently in the same browser instance costs ~50MB extra RAM vs 300MB for a second browser. Safe on 16GB.
- **SearXNG is the highest-ROI next improvement**: Self-hosted, free, replaces "LLM guessing URLs" with "LLM choosing from real search results." Recommended by both Co-CEO and Backend Expert.
- **Browser diversity (Firefox/WebKit) is not needed yet**: No evidence of Chromium detection. `playwright-extra` stealth plugin is the cheaper first step if needed.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| OpenAI 400: `max_tokens not supported` | gpt-5.4-mini requires `max_completion_tokens` | Changed parameter name in `llm-extract.ts` |
| Gemini CLI `spawn gemini ENOENT` | `gemini` binary not in child process PATH | Added explicit path `/home/kos/.npm-global/bin/gemini` and PATH env override |
| Firecrawl 401 on all MCP tool calls | `FIRECRAWL_API_KEY` not in MCP server process env | Updated `.bashrc` to source `op-env-cached.sh`; added `OPENAI_API_KEY` to `.mcp.json` |
| Protocol store garbage patterns | Phase 1 keyword extraction produced low-quality values that were stored as patterns | Cleared `protocols/sites.json`, fixed merge order so LLM overrides stored patterns |
| `tsc` error: `Cannot find name 'DomainConfig'` | Missing import in `pipeline.ts` after adding `overrideConfig` parameter | Added `DomainConfig` to types import |

## Final State
- **14 MCP tools**: submit, status, results, list, domains, cancel, approve, protocols, schedules, schedule_create, auto, diff, timeline, protocols
- **Fetch strategy**: Playwright-first (free) → Firecrawl fallback (paid) → protocol store learns
- **Extraction**: LLM (gpt-5.4-mini → gemini-3-flash) → protocol store → keyword matching
- **Security**: URL sanitization (SSRF blocked), job timeout (120s), tiered agent access
- **Performance**: Parallel Playwright (batches of 2), 12K char LLM context
- **CLI**: Full feature parity with MCP including `auto` command
- **Test results**: Vector databases query → 7/7 sites, 34s, 12-14 fields per site

## Open Questions
- P2 items still pending: query deduplication, cost ledger, queue rotation, approval caller verification, reliability-based site skipping
- SearXNG integration: Co-CEO recommended as highest-ROI improvement (~half day, self-hosted Docker)
- Firecrawl subscription: with Playwright handling most sites for free, evaluate whether to downgrade/cancel
- `playwright-extra` stealth plugin: install when Chromium detection becomes a real problem
- Protocol store pattern quality: need 50+ runs of data to evaluate whether learned regex patterns actually save LLM calls
