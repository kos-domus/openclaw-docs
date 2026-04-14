---
title: "Deep Research Pipeline"
slug: "deep-research-pipeline"
category: "guides"
tags: ["deep-research", "mcp", "automation", "playwright", "firecrawl", "llm"]
sources:
  - "sessions/2026-04-08-deep-research-pipeline-phase1-build.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
  - "sessions/2026-04-08-deep-research-glm51-zai-integration-4.md"
  - "sessions/2026-04-08-deep-research-p2-billing-gemini-testing-3.md"
  - "sessions/2026-04-09-deep-research-vision-ocr-glm-supermarket-test.md"
last_updated: "2026-04-14"
version: 1
---

# Deep Research Pipeline

This guide explains how the internal deep-research stack evolved into a reusable OpenClaw research tool with MCP access, a CLI, protocol learning, and vision support.

## What the pipeline does

The pipeline turns a query or domain config into structured research results:

1. Discover or load target sites.
2. Fetch pages with Playwright first and Firecrawl as fallback.
3. Extract structured fields with subscription-backed LLMs.
4. Store results, history, diffs, and learned fetch patterns.
5. Expose the workflow through MCP tools and a local CLI.

## Recommended architecture

```text
Agent or CLI
  -> deep-research MCP server / CLI
  -> queue + access control
  -> fetchers (Playwright first, Firecrawl fallback)
  -> extractors (GLM-5.1 or Gemini, with keyword fallback)
  -> protocol store + result history
```

## Build sequence that worked

### Phase 1

Start with a minimal but complete vertical slice:

- TypeScript project with typed job/config/result models
- MCP server with submit, status, results, list, cancel, approve, and domain discovery
- CLI wrapper for direct runs
- File-based queue
- Domain YAML configs
- Firecrawl-based fetching

This was enough to prove that multi-site parallel research worked before adding browser rendering or learned extraction.

### Phase 2 and 3 hardening

The pipeline became production-usable after adding:

- Protocol store for site-specific learned patterns and reliability tracking
- Scheduling scaffolding
- LLM-based structured extraction
- Playwright browser fetching with parallel contexts
- `research auto` for plan plus scrape in one call
- Result diffs and timelines
- SSRF protection and job timeouts
- Deduplication, queue rotation, and approval verification
- Vision OCR for image-heavy pages

## Fetch strategy

Use this order by default:

1. **Playwright** for normal pages, because it is local and effectively free.
2. **Firecrawl** when the page is too JS-heavy or Playwright returns thin content.
3. **Vision OCR** when the real data is inside images, flyers, or scanned documents.

### Why Playwright first

It cut costs and still handled many real-world sites. Parallel batches of two pages in one Chromium instance improved throughput without large RAM spikes.

### When to force vision

A page can contain thousands of characters of useless navigation text while the actual data lives inside images. For those sites, add a per-site `forceVision: true` flag instead of relying on content-length heuristics.

## LLM provider strategy

The most cost-aware pattern documented in the sessions was:

1. **Z.AI GLM-5.1** on the coding endpoint when available
2. **Gemini CLI** on OAuth subscription billing
3. **OpenAI API** only as a last-resort fallback

Important details:

- Z.AI has separate coding and regular endpoints, and only the coding endpoint is covered by the Coding Pro subscription.
- GPT-5.x style APIs require `max_completion_tokens`, not `max_tokens`.
- Gemini CLI is good for rich extraction but can intermittently hit capacity errors.

## MCP tool surface

By the end of the processed sessions, the pipeline had tooling for:

- submit and approve jobs
- inspect status and results
- list configured domains
- create and inspect schedules
- run auto-research
- diff results against previous runs
- inspect research history over time

Keep the MCP interface and CLI feature parity aligned. One of the recurring issues was functionality existing in MCP but not in the CLI.

## Security baseline

Do not ship a research fetcher without these protections:

- URL sanitization against private IPs, loopback, metadata endpoints, and dangerous schemes
- explicit job timeouts
- access tiers for agents
- caller verification on approvals
- atomic queue writes

## Best practices

- Prefer domain YAML configs for repeatable research.
- Keep scheduling disabled by default until the use case is clear.
- Use protocol learning to reduce repeated expensive extraction.
- Treat search as a separate concern. The sessions strongly recommended SearXNG as the next high-ROI improvement.
- Store result histories so agents can diff changes instead of re-reading raw outputs every time.

## Related docs

- [Deep Research Reference](../reference/deep-research-reference.md)
- [MCP Servers and Tool Integration](../reference/mcp-servers.md)
- [Deep Research Pipeline Issues](../troubleshooting/deep-research-pipeline-issues.md)
