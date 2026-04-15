---
title: "Built Conad weekly flyer parser and automated PDF discovery"
date: "2026-04-15"
author: "kos-domus"
status: "ready"
tags: ["automation", "configuration", "testing"]
openclaw_version: ""
environment:
  os: "Linux 6.17.0-20-generic"
  ide: "VSCode"
  model: "claude-opus-4-6"
---

## Objective
Add support for Conad weekly store-specific flyers (volantini) to the SpesaBot pipeline — both a regex parser for the PDF text and automated URL discovery from conad.it.

## Context
SpesaBot already had a Conad parser, but only for the "Bassi e Fissi" page (national everyday low prices, ~746 products). Weekly store-specific flyers were noted as "require login, not yet automated" in stores.yaml. The user shared a 30-page PDF flyer for Spazio Conad Bussolengo (17-29 April 2026) and wanted full pipeline support.

Existing parsers for reference:
- **Eurospin**: PDF download + `pdftotext -raw` + regex parser
- **Despar**: Playwright iPaper navigation + text extraction + regex parser
- **Aldi**: Playwright page render + regex on structured markdown

## Steps Taken

### 1. Analyzed the Conad flyer PDF structure
Read all 30 pages of the PDF. Identified 5 distinct product patterns:
- **Discounted products** (pages 1-11): `X,YY€ -XX% € Y,YY` with optional `anziché €/kg` unit prices
- **Fixed-price products**: `X,YY € €/kg Z,ZZ` (no discount)
- **Deli counter items** (page 12): `X,YY € al kg` (per-kg pricing)
- **Percentage-only discounts**: `XX SCONTO %` (no specific price)
- **Quality badges** like "BROCCOLI ITALIANI" or "LATTE ITALIANO" that appear between price blocks and break regex matching

**Result**: Clear understanding of text patterns and edge cases.

### 2. Built the regex parser (`conad-flyer.ts`)
Created `spesabot/src/parsers/conad-flyer.ts` with a multi-pass approach:
- **Pass 1**: Discounted products via `(\d+[.,]\d{2})\s*€\s*-(\d+)\s*%\s*€\s*(\d+[.,]\d{2})` anchor
- **Pass 2**: Products with `anziché €/kg` + separate discount line
- **Pass 3**: Fixed-price products via `(\d+[.,]\d{2})\s*€\s*€/(kg|l)\s+(\d+[.,]\d{2})`
- **Pass 4**: Deli counter via `(\d+[.,]\d{2})\s*€?\s*\nal\s+kg`
- **Pass 5**: Percentage-only discounts via `(\d{2})\s*\n\s*SCONTO\s*%`

Pre-cleaning strips "FINO AL 50%" splash graphics and quality badge lines that appear between price blocks.

The `extractProductFromContext()` function reads backwards from each price anchor to find the product name, brand, and quantity.

**Result**: 14/14 sanity checks pass on inline test data.

### 3. Tested against the real PDF via pdftotext
Created an end-to-end test that downloads the actual PDF from conad.it, runs `pdftotext -raw`, and parses the output.

**Result**: 119 products extracted (74 discounted, 45 fixed-price), dates correctly parsed as 2026-04-17 to 2026-04-29.

### 4. Wired parser into the ingest pipeline
- Added `parseConadFlyerText` import and `'conad-flyer'` key to `CHAIN_PARSERS` in `ingest.ts`
- Added DB slug mapping: `chainSlug.replace(/-flyer$/, '')` so `conad-flyer` resolves to the `conad` DB chain
- Added `'conad-flyer'` to `CHAINS_WITH_PARSER` set in `runner.ts`

**Result**: Clean TypeScript compilation.

### 5. Researched Conad flyer URL discovery options
Two parallel research agents investigated:
- **Conad.it direct**: Store pages at `conad.it/ricerca-negozi/{store}` load flyer links via JavaScript. Static HTML has no PDF URLs.
- **Aggregators** (DoveConviene, VolantinoFacile, PromoQui): All owned by ShopFully. They list flyer IDs but serve page images (not PDFs) and use JS-rendered viewers.
- **Conad PDF hosting**: PDFs live at `conad.it/assets/common/volantini/{coop}/{subfolder}/{filename}.pdf` but subfolder/filename patterns aren't predictable.

Key finding: The cooperative code for Bussolengo is `cia` (not `dao` as initially guessed from the cooperative map).

**Result**: Playwright + persistent browser profile (existing `login-setup.ts`) is the best approach.

### 6. Built the URL discovery module (`conad-flyer-discovery.ts`)
Created `spesabot/src/parsers/conad-flyer-discovery.ts`:
- Uses persistent Chromium profile from `~/.spesabot/browser-profile/`
- Navigates to the store page on conad.it
- Clicks the "Volantini" tab (JS-rendered)
- Intercepts network requests matching `/volantini.*\.pdf/`
- Strips `/renditions/previewvol.webp` suffix to get real PDF URLs
- Filters out non-grocery PDFs (pet catalogs, payment manuals, tax docs, garden furniture, wrong-store flyers)
- Extracts validity dates from filenames (e.g. `17APR_29APR`)

**Result**: Discovers 2 flyers for Bussolengo — the expiring one (7-16 April) and the current one (17-29 April). Fully automated, no manual URL input needed.

### 7. Wired discovery into the runner pipeline
Added a `conad-flyer` handler block in `runner.ts` (parallel to the existing `despar` and `famila` blocks):
- Iterates over configured stores
- Calls `discoverConadFlyers(storeSlug)` for each
- Downloads each PDF via `fetchConadFlyerText()`
- Ingests via `ingestFlyerText()`

**Result**: Full pipeline integration, clean compilation.

## Configuration Changes
- `spesabot/configs/stores.yaml`: Added `conad-flyer` entry with `strategy: pdf`, `parser: conad-flyer`, and Bussolengo store config
- `spesabot/src/ingest.ts`: Added `conad-flyer` to CHAIN_PARSERS, DB slug mapping via `replace(/-flyer$/, '')`
- `spesabot/src/runner.ts`: Added `conad-flyer` to CHAINS_WITH_PARSER and main loop handler

## Key Discoveries
- Conad flyer PDFs are served from `conad.it/assets/common/volantini/{coop}/` — the cooperative for Bussolengo (Verona) is `cia`, not `dao`
- The store page loads flyer cover images as `.pdf/renditions/previewvol.webp` requests — stripping the rendition suffix gives the actual PDF URL
- Quality badges like "BROCCOLI ITALIANI" and "LATTE ITALIANO" appear as separate text lines between a product's unit price and discount percentage, breaking naive regex matching — solved by pre-stripping these patterns
- `\w+` in JavaScript regex does NOT match accented characters (VENERDÌ, MERCOLEDÌ) — must use `\S+` for Italian day names
- The persistent browser profile at `~/.spesabot/browser-profile/` (set up on 2026-04-10) still has a valid Conad session
- DoveConviene, PromoQui, and VolantinoFacile are all owned by ShopFully and share the same backend/flyer IDs — but they serve page images, not PDFs with text layers

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| Date parsing returned null | `\w+` in regex doesn't match accented chars like VENERDÌ | Changed to `\S+` for day name matching |
| BROCCOLI product not extracted | "BROCCOLI ITALIANI" quality badge between price and discount broke regex | Pre-strip quality badge lines from text before parsing |
| YOMO false match (KEFIR consumed YOMO's discount) | Flexible `[\s\S]{0,80}?` bridging allowed regex to jump across products | Reverted to strict `\s*` bridging + targeted badge stripping instead |
| "LAVAGGI" and "SCONTO" extracted as product names | Parser noise from section labels | Added skip filter for known noise words |

## Final State
Complete automated pipeline for Conad weekly flyers:
1. `discoverConadFlyers("spazio-conad-bussolengo")` → finds 2 PDF URLs with dates
2. `fetchConadFlyerText(url)` → downloads PDF, extracts text via pdftotext
3. `parseConadFlyerText(text)` → 119 products with price, discount, brand, quantity
4. Pipeline runner handles the full flow automatically

New files: `conad-flyer.ts`, `conad-flyer-discovery.ts`, `conad-flyer.test.ts`, `conad-flyer-e2e.test.ts`

## Open Questions
- Should the pipeline run `conad-flyer` alongside `conad` (bassi-e-fissi) in the same weekly run, or should they be separate schedules?
- The "PRODOTTI ALLA CARTA" flyer (loyalty-card-only offers) was filtered as wrong-store — should it be included with a different parser?
- Adding more Conad stores beyond Bussolengo would require adding their store page URLs to the `STORE_PAGES` map in `conad-flyer-discovery.ts`
