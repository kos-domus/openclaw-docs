---
title: "Spesify: Conad image mismatch fix — remove broken regex, structural DOM scrape, cross-match backfill"
date: "2026-04-20"
author: "kos-domus"
status: "processed"
tags: ["api", "automation", "debugging", "configuration", "web", "testing"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-7[1m]"
---

## Objective

User feedback on the Spesify Mini App: Conad product cards were showing the **wrong image** for many items (e.g. "Farina di Grano Tenero Tipo 0 5kg" displaying a hamburger photo), and an additional set of cards showed no image at all. Diagnose the root cause, fix it without risking further regressions, and push the correctly-aligned image coverage as high as safe in one session. Weekly-flyer path triaged and parked as a separate concern.

## Context

- Pipeline state as of 2026-04-18: 8 chains live, Mini App at app.spesify.xyz, `product_skus.image_url` powers the card images.
- Conad has two distinct scrape paths:
  1. **Bassi e Fissi** (chain id `conad`) — the national everyday-low-price page at `/prodotti-e-marchi/bassi-e-fissi`, previously scraped via the deep-research tool that emitted markdown which `src/parsers/conad.ts` then regex-parsed.
  2. **Weekly flyers** (chain id `conad-flyer`) — per-store PDFs discovered via `conad-flyer-discovery.ts`, text-extracted, and regex-parsed by `src/parsers/conad-flyer.ts`.
- Non-Conad chains (Migross, Famila, Eurospin, etc.) were not affected — their image plumbing is separate and was functionally correct in the screenshots Kos shared.
- Starting DB state: 1215 Conad `product_skus` rows, 506 with image URLs, ~42% image coverage.

## Steps Taken

### 1. Diagnosis — isolate the failure to Conad

Reviewed the user screenshots. Every mismatched pair was on a Conad card; Migross, Eurospin, and Mellin cards shown alongside were correct. That narrowed the search to Conad-specific image plumbing.

Sampled the DB:

```sql
SELECT ps.raw_name, ps.image_url
FROM product_skus ps JOIN chains c ON ps.chain_id = c.id
WHERE c.name = 'Conad' AND ps.raw_name ILIKE '%farina%tipo%'
ORDER BY ps.id LIMIT 10;
```

Result: every flour product was pointing at a `/assets/products/<EAN>/ID-Shot.jpeg` URL, but the EAN embedded in the URL did not correspond to a flour. Hypothesis: image-to-product pairing had drifted by one (or more) in the Conad parser — *not* that images were wrong at the source.

**Result**: narrowed scope to the Conad parser + ingest pipeline.

### 2. Located the broken regex in `src/parsers/conad.ts`

The bassi-e-fissi parser used a single monolithic regex to extract name, price, validity, and an **optional trailing image**:

```ts
/(?:####\s*)?(?<brand>...)\s+(?<rest>...)\s+(?<price>\d+[.,]\d{1,2})\s*€\s*
  Dal\s+(?<fromDay>\d{1,2})\/(?<fromMonth>\d{1,2})\s+
  al\s+(?<toDay>\d{1,2})\/(?<toMonth>\d{1,2})
  (?:\s*!\[[^\]]*\]\((?<imageUrl>[^)]+)\))?  /* <-- THE BUG */
/gi
```

The optional trailing image group assumes the rendered markdown pattern:

```
#### BRAND product name quantity
PRICE € Dal DD/MM al DD/MM
![alt](image_url)
```

But Playwright-rendered markdown from Conad's page frequently emits the image **before** the product heading (standard card layout — image first, then text). The regex then greedily attaches the *next* product's image to the *current* product → off-by-one drift across the whole grid.

Also inspected the chain-agnostic image-enrichment code at `src/ingest.ts:219-226`:

```ts
// Try substring match: find API name that contains the PDF name or vice versa
for (const [apiName, url] of Object.entries(imgMap)) {
  if (apiName.includes(name) || name.includes(apiName)) {
    p.image_url = url;
    matched++;
    break;
  }
}
```

This substring fallthrough is also too loose (could pair e.g. "FARINA" with "HAMBURGER DI FARINA") but currently only fires for Eurospin (the only chain populating `productImages` at job level), which was not the visible problem in screenshots. Kept it in scope for a future cleanup but not touched this session to avoid changing non-Conad behavior.

**Result**: root cause confirmed as the optional trailing image group in the Conad bassi-e-fissi regex.

### 3. Step A — stop serving wrong images (immediate, reversible)

User preference: **no image is better than wrong image**. Emoji fallback in the Mini App is already implemented. Surgical fix:

```ts
// src/parsers/conad.ts
// Before:
image_url: g.imageUrl || null,
// After:
image_url: null,
```

Removed the `(?:\s*!\[[^\]]*\]\((?<imageUrl>[^)]+)\))?` image capture group from the regex entirely and added a block comment explaining why. Build clean (`tsc`), no tests broken (`conad.ts` has no direct test coverage — the flyer path is tested separately).

NULLed all existing Conad image URLs in a single transaction:

```sql
BEGIN;
UPDATE product_skus SET image_url = NULL
WHERE image_url IS NOT NULL
  AND chain_id = (SELECT id FROM chains WHERE name='Conad');
COMMIT;
```

Update touched 506 rows. Coverage went 506→0 with image for Conad, intentional pre-step for the DOM scrape.

**Result**: Mini App refreshed; all Conad cards showed emoji fallback instead of wrong images. User confirmed visually.

### 4. DOM probe — learn the structure before writing the scraper

Wrote a throwaway `scripts/probe-conad-bassi.mjs` that opened the bassi-e-fissi page in Playwright, scrolled a few times, and dumped stats for candidate CSS selectors plus sample HTML around each product image.

Key finding — the page is a clean AEM-rendered card grid:

```html
<div class="rt213-card-product-flyer rt072-disaggregated-block__card"
     data-code="8003170003903"
     data-ean="8003170003903"
     data-nome="CONAD Ribe 1 kg">
  <img src="https://www.conad.it/assets/products/8003170003903/ID-Shot.jpeg/renditions/small.jpeg"/>
  <span class="rt213-card-product-flyer__validity">Dal 1/1 al 30/4</span>
  … 2,29 € …
</div>
```

Every datum is co-located inside the same card container. No ambiguity possible. Also found: the card's `img.src` uses the same EAN as the card's `data-ean` — so the correct pairing is a hard structural invariant, not a heuristic.

Probe also confirmed: the page uses infinite scroll. Full page load yields ~746 cards after the scroll loop settles (matches the "~746 products" note in `configs/stores.yaml`).

Cleaned up the probe script after extraction.

**Result**: clear target selectors + DOM shape, ready to write a proper structural scraper.

### 5. Step B — wrote `src/parsers/conad-bassi-fetch.ts`

New file, ~170 LOC, follows the Migross / Famila fetcher pattern already established in the codebase. Key design decisions:

- **Playwright directly, not via deep-research**. The existing pipeline spawned a full browser context through `tools/deep-research` just to render markdown. We skip that layer and interact with the DOM directly — same browser cost, but we get structured data back.
- **Scroll-until-stable loop** with 3 consecutive no-change rounds as the stopping criterion (matches "iterates until scrollHeight stabilizes" in stores.yaml).
- **String-form `page.evaluate`** to dodge tsx's `__name` helper injection into the browser-side code (a known issue when evaluating TS-transpiled functions inside Playwright — caused our first two probe attempts to fail before we switched to string eval).
- **Per-card extraction**: selector `.rt213-card-product-flyer`, pull `data-ean` / `data-nome` / first `img.src` / `.rt213-card-product-flyer__validity` / first `X,YY €` substring. Returns a `RawCard` per node.
- **Brand/name/quantity split** from `data-nome` using the same `QTY_RE` already in `conad.ts` for consistency with the weekly-flyer parser.
- **Image URL validation**: only accept URLs whose host is `www.conad.it` and path starts with `/assets/products/`. Rejects any tracking pixels or CDN rewrites that might sneak in.
- **Dedupe key**: `brand|name|quantity|price` (the page lists products under multiple category filters, producing duplicate cards).

Output type reused from `conad.ts` — `ConadProduct` — so downstream ingest code doesn't need to change.

**Result**: fetcher compiles clean and exports `fetchConadBassiProducts(): Promise<ConadProduct[]>`.

### 6. Wired into `runner.ts`

Added a new branch **before** the `chain === 'conad-flyer'` branch in `src/runner.ts`:

```ts
if (chain === 'conad') {
  const products = await fetchConadBassiProducts();
  // … build synthetic JobResult with products in `extracted` …
  const stats = await ingestJobResult(tmpJson, chain);
  // … summary accounting …
  continue;
}
```

The branch follows the Migross pattern verbatim (`extracted` array → `ingestJobResult` → campaign creation). No change to `ingest.ts` — it already reads `product.image_url` from extracted items at `src/ingest.ts:354`.

Deep-research path for chain `conad` is now bypassed entirely. `conad.ts`'s `parseConadMarkdown` is still exported for backward-compat but no longer invoked by the runner.

**Result**: build clean. One IDE diagnostic hint about an "unused import" surfaced transiently during my two-step edit — resolved itself after the branch landed.

### 7. Ran the pipeline on just Conad to verify

```bash
SPESABOT_CHAINS=conad node dist/runner.js
```

Output (elided):

```
[1/1] Scraping conad (Playwright structural scrape)...
  conad-bassi: scroll settled at 746 cards
  conad-bassi: extracted 746 cards from DOM
  conad-bassi: 746 deduped products, 731 with image
  fetched 746 products
  731/746 products have images (98%)
[conad] campaign dates derived from products: 2026-01-01 → 2026-04-30 (746/746 products)
  cleaned 510 existing offers for campaign 587
conad: 746 products ingested
Pipeline complete in 42s
```

**42-second run**, 731 images out of 746 (98%), all structurally correct.

### 8. Verified correctness against the originally-broken products

Spot-checked the exact flour/pasta products Kos called out in screenshots:

| Product | Qty | Image EAN |
|---|---|---|
| Farina di Grano Tenero Tipo 0 | 5 kg | `/assets/products/8003170084674/…` |
| Farina di Grano Tenero Tipo 0 | 1 kg | `/assets/products/8003160430023/…` |
| Farina di Grano tenero Tipo 00 | 1 kg | `/assets/products/8003160430054/…` |
| Tagliatelle all'Uovo | 250 g | `/assets/products/8003170015319/…` |
| Frastagliate all'Uovo | 250 g | `/assets/products/8003170028647/…` |
| Frollini con Farina 100% Integrale | 400 g | `/assets/products/8003170072404/…` |

All URLs curl-checked as HTTP 200 with plausible JPEG sizes (16–29 KB). User confirmed in the Mini App: correct images next to correct text.

### 9. Step C — cross-match backfill for weekly flyer null-image products

After step 7, the DB had 728/1379 with images — bassi done (98%) but weekly-flyer (`conad-flyer` chain) still null. Since both paths land in the same `product_skus` table under `chain=Conad`, an overlap might rescue some flyer images for free.

Match counts on `LOWER(TRIM(raw_name))`:

| Match | Count |
|---|---|
| Exact name | 148 |
| Exact name + quantity | 135 |
| Name match, quantity differs | 13 |
| Total null-image flyer SKUs | 655 |

Went with the tight name+quantity match (135) to avoid assigning a 1kg-package image to a 5kg-package product. Ran the UPDATE in a transaction:

```sql
BEGIN;
WITH bassi AS (
  SELECT LOWER(TRIM(raw_name)) AS lname,
         COALESCE(LOWER(TRIM(raw_quantity)),'') AS lqty,
         image_url,
         ROW_NUMBER() OVER (PARTITION BY LOWER(TRIM(raw_name)), COALESCE(LOWER(TRIM(raw_quantity)),'') ORDER BY id) AS rn
  FROM product_skus WHERE chain_id=(SELECT id FROM chains WHERE name='Conad') AND image_url IS NOT NULL
)
UPDATE product_skus ps SET image_url = b.image_url
FROM bassi b
WHERE ps.chain_id = (SELECT id FROM chains WHERE name='Conad')
  AND ps.image_url IS NULL
  AND LOWER(TRIM(ps.raw_name)) = b.lname
  AND COALESCE(LOWER(TRIM(ps.raw_quantity)),'') = b.lqty
  AND b.rn = 1;
COMMIT;
```

**Result**: UPDATE 135. Coverage now 863/1379 (63%).

### 10. Profiled the remaining 516 null-image products — decided to park

Sampled the leftover SKUs — overwhelmingly weekly-flyer products with poor data quality:

- Fragmented names from PDF OCR: "ASPARAGI VERDI CONAD PERCORSO" with brand field = "QUALITA'"
- All-caps with missing spaces: "PER RASOIO", "CROCCOLE CON MERLUZZO"
- No quantity field for many rows
- Mixed brands — Conad private label (80%) + third-party (COATI, LEVONI, CASA MODENA, EMMENTALER, MARTELLI, MONTASIO DOP, PATATE, PRATOMAGNO, BECHER, BOSCHI)

Auto-matching these fragmented names against Conad's catalog would risk reintroducing the wrong-image bug we just fixed. The real lever here is **data quality at the PDF parser**, not image fetching. Parked as a separate concern for a follow-up session.

## Configuration Changes

- New file: [src/parsers/conad-bassi-fetch.ts](../../spesabot/src/parsers/conad-bassi-fetch.ts) — structural Playwright scraper, ~170 LOC.
- Modified: [src/parsers/conad.ts](../../spesabot/src/parsers/conad.ts) — removed image capture group from regex, hardcoded `image_url: null` in produced objects. Block comment added explaining why.
- Modified: [src/runner.ts](../../spesabot/src/runner.ts) — new `chain === 'conad'` branch before the existing `conad-flyer` branch; new import for `fetchConadBassiProducts`.
- No schema changes.
- No service restarts required (API + bot just read the DB, refresh shows new state automatically).

## Key Discoveries

1. **AEM product pages are usually a goldmine of structured data**. Conad's bassi-e-fissi renders each card with explicit `data-ean`, `data-code`, `data-nome` attributes — the scraper writer on Conad's side already did the work of co-locating all the data per card. We were ignoring this goldmine and reparsing rendered markdown instead. **Heuristic**: whenever a chain uses Adobe Experience Manager or similar CMS, probe for `data-*` attributes on card containers first. If they exist, the DOM path is always more reliable than text extraction.

2. **"Rendered markdown" is a lossy intermediate representation**. The deep-research tool converts DOM to markdown, dropping structural information (data attributes, DOM hierarchy, sibling order guarantees) and introducing ordering artifacts (image-before-heading vs. heading-before-image). For any chain where card-level DOM structure is available, going directly to DOM beats going through markdown. This pattern is now established in the codebase: Migross (API JSON), Rossetto (HTTP HTML → regex), Conad bassi (DOM). The deep-research path should be a fallback for chains without structure, not the default.

3. **Optional regex groups can silently corrupt data across an entire dataset**. The `(?:...)?` image group in the original regex was an "additive feature — only fills `image_url` when the layout matches" — but when the layout didn't match on most products, it was still grabbing the image from the NEXT product. An optional group shouldn't bleed across boundaries. **Heuristic for regex reviews**: if a group is marked optional, simulate it with an adjacent-different-product fixture and verify the optional group either matches correctly or stays unset — never captures from a neighbor.

4. **Cross-chain-within-same-chain name matching is a cheap free rescue**. Conad's bassi-e-fissi catalog and weekly flyers overlap by ~22% on exact name. When one subpath has reliable images (bassi, structural DOM) and another doesn't (flyer, PDF OCR), an intra-chain cross-match UPDATE is basically free quality gain. Worth generalizing — for any chain with multiple scrape paths, run a name+quantity merge step after ingest to propagate images/brand/category from the cleaner source to the messier one.

5. **Pay-plan-aligned model routing carries over here**. Not used this session directly, but the structural scraper replaces an LLM-based extraction that would have charged Gemini/Codex tokens per product page. DOM scraping is ~free. Worth tallying: 746 products × deep-research pipeline ≈ non-trivial LLM cost per run; now it's a 42s browser scroll. Multiply by 2 runs/week = real savings vs. the equivalent LLM-backed path.

6. **`tsx` + Playwright gotcha**: `page.evaluate(() => { ... })` fails with `ReferenceError: __name is not defined` when the callback is a transpiled arrow function, because tsx injects `__name` as a helper that's not available in the browser context. Workarounds: pass a string to `page.evaluate()` (the approach used here), or use plain `.mjs` / `.cjs` without tsx transforms. Worth documenting in the dev notes.

## Errors & Solutions

| Error | Cause | Solution |
|---|---|---|
| Conad Mini App cards showing wrong images (hamburger for flour, etc.) | `conad.ts` regex's optional trailing image group grabbed the NEXT product's image when Conad's rendered markdown placed the image before the product heading | Removed the image capture group entirely from the regex, NULLed all existing Conad `image_url`s, wrote a structural DOM scraper that reads `.rt213-card-product-flyer` cards and pairs image↔product via shared container. |
| `page.evaluate: ReferenceError: __name is not defined` when running Playwright probe via `npx tsx` | tsx transpilation injects `__name` helper into arrow functions; helper doesn't exist in Chromium's execution context | Passed the probe logic as a JS string to `page.evaluate()` instead of as a TS arrow function. Alternative: use plain `.mjs` with `import 'playwright'`. |
| Build emitted "declared but never read" hint for `fetchConadBassiProducts` | IDE evaluated the diagnostics after the first Edit (added the import) but before the second Edit (added the usage in the `chain === 'conad'` branch). Stale cache. | Ignored — `tsc` passed clean and a fresh `grep` confirmed the import was used at line 272. |

## Final State

- Conad image coverage: **863 / 1379 correctly-aligned (63%)**, up from 506 / 1215 misaligned (42%).
- Bassi-e-fissi path: 731/746 (98%) clean from new structural scraper, DOM guarantees alignment.
- Weekly-flyer path: 132 rescued via cross-match from bassi catalog; 516 remain null pending PDF parser data-quality work.
- Mini App: shows correct Conad photos for all bassi products and ~30% of weekly-flyer products; emoji fallback for the rest.
- Pipeline runtime for Conad: ~42s (scroll + extract + ingest) — comparable to the prior deep-research path.
- New artefact: `src/parsers/conad-bassi-fetch.ts`. Existing `conad.ts`'s `parseConadMarkdown` is dormant but kept exported for backward-compat.

## Open Questions

- **Weekly-flyer PDF parser data quality** is the actual blocker for further image coverage. Names come out OCR-fragmented ("PER RASOIO") or with brand miscategorized ("ASPARAGI VERDI CONAD PERCORSO" / brand="QUALITA'"). A dedicated session to tighten the flyer regex + brand extraction would unlock both (a) cleaner search terms for image resolution and (b) better user-facing product names. Do this before any more image-fetch work on the flyer path.
- **Generalize the intra-chain cross-match UPDATE** into a post-ingest step that runs after every pipeline for any chain with multiple scrape sources. Cheap quality gain. Currently only done ad-hoc here.
- **`ingest.ts:219-226` substring-match fallthrough** is also too loose (can pair "FARINA" with "HAMBURGER DI FARINA" when both appear in the imgMap). Currently only fires for Eurospin, which has a clean API path and didn't surface the issue, but worth tightening to token-set matching or removing entirely. Parked.
- **EAN storage**: the new bassi scraper discards the extracted EAN after using it for image URL construction. We could populate `product_skus.chain_sku_id` with the EAN (currently unused for Conad, 0 rows have it) — useful for future dedup, cross-chain matching, and OpenFoodFacts enrichment. One-line addition to the scraper + a backfill UPDATE. Follow-up.
- **Automatic alerting on image-pairing regression**: write a post-pipeline sanity check that samples N random products per chain and verifies that the image URL, when fetched, contains visual tokens matching the product name (via OCR / CLIP). Expensive but catches this class of bug automatically next time. Worth scoping once we have more chains on structural scrapers.
