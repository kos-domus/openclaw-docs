---
title: "Deep Research — Vision OCR for image-heavy sites, supermarket flyer test, forceVision config"
date: "2026-04-09"
author: "kos-domus"
status: "ready"
tags: ["mcp", "mcp-servers", "automation", "configuration", "api", "troubleshooting"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Add vision-based OCR extraction to the deep research pipeline for image-heavy sites (supermarket flyers, PDFs rendered as images), test with real Italian supermarket deals from doveconviene.it, and increase default job timeout.

## Context
- Pipeline built over previous sessions with 14 MCP tools, Playwright + Firecrawl fetching, GLM-5.1 + Gemini + OpenAI LLM stack
- Kos tested supermarket deal research: doveconviene.it pages returned HTML but actual deals are in flyer images
- Pipeline extracted almost nothing because the data was in images, not text
- Need: screenshot capture + vision LLM to read image-based content

## Steps Taken

### 1. Tested supermarket deals — identified the problem
Ran research on doveconviene.it for Lidl, Eurospin, Famila, Martinelli. Pages returned 3,500+ chars of HTML but it was all navigation links — actual product/price data is in scanned flyer images embedded in the page.

**Result**: Pipeline extracted only "Scade mercoledì" and "60%" — no products, no prices, no brands.

### 2. Investigated vision model availability on Z.AI plan
Tested which vision models are included in Kos's Z.AI Coding Pro subscription:

| Model | Available | Notes |
|-------|-----------|-------|
| GLM-5V-Turbo | No | "Plan does not include access" |
| GLM-4.6V | Yes | Full vision model |
| GLM-4.6V-Flash | Yes | Fast vision model |
| GLM-4.5V | Yes | Older vision model |

**Result**: GLM-4.6V and GLM-4.6V-Flash available on coding plan at zero extra cost.

### 3. Built vision OCR extraction
Added `llmVisionExtract()` function to `llm-extract.ts`:
- Takes a base64 JPEG screenshot + domain schema
- Sends to `glm-4.6v-flash` via Z.AI coding endpoint
- Prompt instructs model to read ALL visible text including prices, percentages, brand names, weights
- Supports array output for pages with multiple items (product lists)
- 60s timeout for vision processing

### 4. Updated Playwright fetcher with screenshot capture
When text content is thin (<500 chars) OR `forceVision` is set:
- Scrolls page to trigger lazy-loaded content
- Takes full-page JPEG screenshot (quality 70)
- Caps screenshot at 2MB base64 (retakes viewport-only at quality 50 if too large)
- Returns screenshot alongside text for the pipeline to route to vision extraction

Kept images loading in Playwright (previously blocked for speed) since screenshots need them.

### 5. Added `forceVision` per-site config
New `forceVision: boolean` field in `SiteConfig` type. When set in domain YAML, forces screenshot capture regardless of text content length.

Updated supermarket-deals.yaml:
```yaml
sites:
  - url: https://www.doveconviene.it/volantino/lidl
    method: playwright
    forceVision: true
```

### 6. Wired vision into pipeline
Pipeline now has three extraction paths:
1. **Vision OCR** — when fetch result includes a screenshot (thin content or forceVision)
2. **Text LLM extraction** — standard path for text-rich pages
3. **Failure recording** — when both fetch and extraction fail

### 7. Increased default job timeout to 300s
Changed from 120s to 300s (`RESEARCH_JOB_TIMEOUT` env var). Multi-site jobs with LLM extraction per site need more time, especially with reasoning models like GLM-5.1 (~35s per extraction).

### 8. Testing blocked by Z.AI 429
All Z.AI models (GLM-5.1, GLM-4.6V-Flash) returning 429 "service temporarily overloaded" throughout the evening. The vision OCR flow triggered correctly (screenshot captured, vision API called), but the API was unavailable.

**Result**: Pipeline is wired and ready. Needs Z.AI to recover for actual vision extraction test.

## Configuration Changes
- `~/job-desk/tools/deep-research/src/fetchers/playwright.ts` — added screenshot capture, `forceVision` parameter, kept images loading
- `~/job-desk/tools/deep-research/src/normalizer/llm-extract.ts` — added `llmVisionExtract()` function with GLM-4.6V-Flash, `zai-vision` provider type
- `~/job-desk/tools/deep-research/src/pipeline.ts` — added vision OCR path in result processing, pass `forceVision` from site config, timeout 120s → 300s
- `~/job-desk/tools/deep-research/src/types.ts` — added `forceVision?: boolean` to `SiteConfig`
- `~/job-desk/tools/deep-research/configs/supermarket-deals.yaml` — added `forceVision: true` on all 4 sites

## Key Discoveries
- **Supermarket deal sites use image-based flyers**: doveconviene.it, volantinofacile.it, promoqui.it all display deals as scanned PDF/JPG flyer images. The HTML text is just navigation and metadata. This is a common pattern for Italian supermarket aggregators.
- **GLM-4.6V and GLM-4.6V-Flash are on the Z.AI Coding Pro plan**: Vision models available at no extra cost. GLM-5V-Turbo is NOT included.
- **Text content length is a poor proxy for "needs vision"**: doveconviene.it returned 3,500 chars of HTML, well above the 500-char threshold, but none of it was product data. The `forceVision` per-site flag is the right solution — you know at config time which sites have image-based content.
- **Z.AI has intermittent capacity issues**: Throughout this session, both text and vision models returned 429s. This is server-side, not rate-limiting. The fallback chain (Z.AI → Gemini → OpenAI) handles this gracefully.
- **Full-page JPEG screenshots at quality 70 are ~500KB-2MB**: Acceptable for vision model input. The 2MB base64 cap prevents oversized requests.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| doveconviene.it returned text but no product data | Deals are in flyer images, not HTML | Added vision OCR with screenshot capture |
| Screenshot not triggered (3,500 chars > 500 threshold) | Text threshold too naive for image-heavy sites | Added `forceVision` per-site config flag |
| Z.AI Vision 429 "service temporarily overloaded" | Z.AI server capacity issue (all models) | Intermittent — pipeline falls through gracefully. Test when recovered. |
| Job timeout at 120s on multi-site jobs | Not enough time for Playwright + LLM per site | Increased default to 300s |

## Final State
- **Vision OCR pipeline**: Built and wired. Playwright screenshots → GLM-4.6V-Flash → structured extraction
- **`forceVision` config**: Per-site flag in domain YAML for image-heavy sites
- **Default timeout**: 300s (was 120s)
- **Supermarket deals config**: 4 sites with forceVision enabled
- **Blocked by Z.AI 429**: All features built, need API recovery for end-to-end test

## Open Questions
- Z.AI 429 reliability: monitor over next days. If persistent, consider adding Gemini vision as fallback (Gemini CLI supports image input)
- Martinelli page on doveconviene.it returns 404 — small local chain may not be listed. Need alternative source.
- Should we add a Gemini vision fallback? Gemini CLI supports `--image` flag but need to test if it works with subscription billing
- For auto-research: when the LLM discovers sites for a query, should it auto-detect which ones need `forceVision`? (e.g., if the URL contains "volantino" or "flyer")
