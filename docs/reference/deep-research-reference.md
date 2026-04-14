---
title: "Deep Research Reference"
slug: "deep-research-reference"
category: "reference"
tags: ["deep-research", "mcp", "cli", "reference", "research"]
sources:
  - "sessions/2026-04-08-deep-research-pipeline-phase1-build.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
  - "sessions/2026-04-08-deep-research-glm51-zai-integration-4.md"
  - "sessions/2026-04-08-deep-research-p2-billing-gemini-testing-3.md"
  - "sessions/2026-04-09-deep-research-vision-ocr-glm-supermarket-test.md"
last_updated: "2026-04-14"
version: 1
---

# Deep Research Reference

## Core components

```text
tools/deep-research/
├── src/types.ts
├── src/config.ts
├── src/pipeline.ts
├── src/mcp-server.ts
├── src/cli.ts
├── src/queue/job-queue.ts
├── src/fetchers/firecrawl.ts
├── src/fetchers/playwright.ts
├── src/normalizer/normalize.ts
├── src/normalizer/llm-extract.ts
├── src/protocol-store.ts
├── src/schedules.ts
├── src/auto-research.ts
├── src/research-memory.ts
└── configs/*.yaml
```

## Fetch order

1. Playwright
2. Firecrawl fallback
3. Vision OCR when screenshot data is present or `forceVision` is enabled

## Important config fields

### Domain config

| Field | Meaning |
|---|---|
| `sites` | List of sources to fetch |
| `method` | Preferred fetch method |
| `forceVision` | Force screenshot plus vision extraction even if text exists |
| `extractionMode` | Single object or array-style extraction |
| `maxItemsPerPage` | Upper bound for product-style array extraction |

### Runtime controls

| Field | Meaning |
|---|---|
| `RESEARCH_JOB_TIMEOUT` | Overall per-job timeout |
| access tiers | Full, standard, or request-level usage |
| reliability threshold | Skip sites below roughly 30 percent after repeated failures |

## Known tool families

### MCP tools

The processed sessions documented these tool families:

- job submission and approval
- status and results retrieval
- domain listing
- schedules
- auto-research
- diffs and timelines
- protocol inspection

### CLI patterns

```bash
research submit ...
research status ...
research results ...
research list
research auto "your question here"
```

If you add an MCP capability, add the CLI equivalent in the same change.

## Provider notes

### Z.AI

- Use the coding endpoint for subscription-backed usage.
- GLM-5.1 behaves like a reasoning model and may need longer timeouts.
- Vision extraction was wired through GLM-4.6V / GLM-4.6V-Flash.

### Gemini CLI

- Use OAuth-backed CLI auth for subscription billing.
- `-p` is the prompt flag.
- Long prompts are safer through stdin piping.
- Capacity errors can happen even when the command is correct.

### OpenAI API

- Keep it as fallback when raw API billing is acceptable.
- GPT-5.x requests require `max_completion_tokens`.

## Protocol store behavior

The protocol store learns per-site fetch and extraction patterns. Use it to cache:

- preferred fetch method
- success and failure counts
- average fetch time
- reliability score
- learned content anchors or extraction hints

When merging learned data with new LLM extraction, the higher-quality LLM result should win.

## Vision OCR notes

Use screenshots for:

- supermarket flyers
- scanned PDF pages
- image-heavy offer aggregators

Typical implementation details from the sessions:

- full-page JPEG screenshot
- base64 payload cap around 2 MB
- viewport-only fallback if the screenshot is too large
- explicit prompt to read prices, percentages, brands, and weights

## Search and planning

`research auto` was designed to:

1. plan a set of authoritative sites
2. define an extraction schema
3. scrape and normalize results
4. optionally save a reusable domain config

For better URL discovery quality, the sessions recommended adding SearXNG instead of relying on LLM URL guessing.
