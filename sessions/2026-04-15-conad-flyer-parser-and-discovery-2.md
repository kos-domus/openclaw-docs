---
title: "Conad flyer: dynamic store discovery via API + all VR stores + loyalty flyer support"
date: "2026-04-15"
author: "kos-domus"
status: "processed"
tags: ["automation", "api", "configuration", "testing"]
openclaw_version: ""
environment:
  os: "Linux 6.17.0-20-generic"
  ide: "VSCode"
  model: "claude-opus-4-6"
---

## Objective
Extend the Conad flyer pipeline (built earlier today) with:
1. Dynamic store discovery via Conad's internal API (replacing hardcoded store URLs)
2. Coverage for all Conad stores in Verona province
3. Support for PRODOTTI ALLA CARTA loyalty card flyers

## Context
The first session built a Conad flyer parser and Playwright-based PDF discovery for a single hardcoded store (Spazio Conad Bussolengo). The user wanted to automate store discovery so new stores are picked up automatically, cover all VR-province stores, and handle the loyalty card flyer format.

## Steps Taken

### 1. Researched Conad flyer discovery options
Two parallel research agents investigated:
- **Conad.it**: Store pages load flyer links via JavaScript only. No public API documented.
- **Aggregators** (DoveConviene, VolantinoFacile, PromoQui): All owned by ShopFully. They serve page images (not PDFs with text layers) and use JS-rendered viewers.

**Result**: Playwright + persistent browser profile on conad.it confirmed as the best approach. Aggregators are image-only — unusable for our `pdftotext`-based pipeline.

### 2. Discovered the Conad store search API
Built a Playwright script that intercepted all network requests on the store locator page. Found a POST API:

```
POST https://www.conad.it/api/corporate/it-it.retrievePointOfService.json
Body: {"latitudine":"45.4384","longitudine":"10.9917","raggioRicerca":"50","insegneId":[],"serviziId":[],"repartiId":[],"apertura":[]}
```

This returns full store data including `pdvPlainUrl`, `nomeComune`, `volantiniCount`, `anacanId`, `codiceCooperativa`, and opening hours.

**Result**: 45 stores within 50km. After filtering to VR province + non-PetStore + non-specialized: **6 grocery stores**.

### 3. Identified all VR-province Conad stores

| Store | City | Code | Coop | Flyers |
|-------|------|------|------|--------|
| Tuday Conad (Via Stella) | Verona | 010521 | DAO | 0 |
| Tuday Conad (Piazzetta Pescheria) | Verona | 010082 | DAO | 0 |
| Conad (Vicolo Fossetto) | Verona | 009113 | DAO | 2 |
| Conad (Via Pisano) | Verona | 009033 | DAO | 2 |
| Spazio Conad | Bussolengo | 009945 | CIA | 9 |
| Conad Superstore | Peschiera del Garda | 009958 | DAO | 3 |

Tuday Conad stores (210 sqm convenience format) never publish their own flyers — included in listing but auto-skipped during discovery.

**Result**: Cooperative code for Bussolengo is `CIA` (Commercianti Indipendenti Associati), not `DAO` as previously assumed. The Verona city stores use `DAO`.

### 4. Refactored discovery to use API-based store listing
Added `listConadStores()` function to `conad-flyer-discovery.ts`:
- Calls the Conad API from within a Playwright page context (needs cookies)
- Filters by `codiceProvincia === 'VR'`, non-specialized, non-PetStore
- Returns `ConadStoreInfo[]` with slug, store page URL, flyerCount

Extracted `openBrowserContext()` helper to eliminate duplicate browser launch code.

Updated `runner.ts` to call `listConadStores()` dynamically instead of reading from `stores.yaml`.

**Result**: New stores will be auto-discovered. No YAML changes needed when stores open/close.

### 5. Fixed city-based flyer filtering
The original filter skipped any flyer whose URL didn't contain the target city name. This broke regional flyers like `AP0826_CONAD_VENETO_seindao.pdf` (no city, shared across Veneto).

Replaced with `isFlierForDifferentCity()` — only skips flyers that mention a **known different city** (from a list of ~17 Italian cities in the cooperative areas).

**Result**: Regional flyers (CONAD_VENETO, Superstore) now pass through. Wrong-store flyers (Mortegliano) still filtered.

### 6. Tested discovery across all 4 stores with flyers

| Store | Flyers Discovered |
|-------|-------------------|
| Conad Verona (Fossetto) | 2: CONAD_VENETO regional + convenienti sempre |
| Conad Verona (Pisano) | 2: same regional flyers |
| Spazio Conad Bussolengo | 3: previous week + PRODOTTI ALLA CARTA + current week |
| Conad Superstore Peschiera | 2: Superstore flyer + convenienti sempre |

**Result**: All stores return flyers. The `PRODOTTI ALLA CARTA` loyalty flyer now passes through.

### 7. Added PRODOTTI ALLA CARTA loyalty flyer support
The loyalty flyer uses a different format:
- Prices as bare `X,YY` (no `€` suffix)
- Standalone percentages `15%` (no `SCONTO` keyword)
- Heavy marketing copy interleaved with product data

Added two new parser passes to `conad-flyer.ts`:
- **Pass 6**: `\n(\d+[.,]\d{2})\s*\n€/(kg|l)\s+(\d+[.,]\d{2})` — bare price + unit price
- **Pass 7**: `\n(\d{1,2})%\s*\n` — standalone percentage on its own line

Also expanded `NOISE_RE` and `NOISE_LINES` for loyalty flyer marketing text (L'Oréal promos, cashback disclaimers, etc.).

**Result**: 5 products extracted from the loyalty flyer. Main weekly flyer improved from 119 to 134 products (new passes caught 15 extras there too). All 14 sanity checks still pass.

## Configuration Changes
- `spesabot/src/parsers/conad-flyer-discovery.ts`: Added `listConadStores()`, `openBrowserContext()`, `isFlierForDifferentCity()`, CLI commands (`stores`, `discover`), `KNOWN_FLYER_CITIES` list, `Parafarmacia` skip pattern
- `spesabot/src/parsers/conad-flyer.ts`: Added Pass 6 (bare price + unit price), Pass 7 (standalone percentage), expanded noise filters, `15%` as context boundary
- `spesabot/src/runner.ts`: Updated `conad-flyer` handler to use `listConadStores()` instead of hardcoded stores, import `listConadStores`

## Key Discoveries
- Conad has a public-ish store search API at `/api/corporate/it-it.retrievePointOfService.json` — POST with lat/lng/radius, returns full store data including flyer counts. Works from a Playwright page context (needs cookies from initial page load).
- The API response includes `volantiniCount` per store — useful for skipping stores with no active flyers.
- Cooperative codes matter: `CIA` (Bussolengo) and `DAO` (Verona city) are different cooperatives under the Conad umbrella, with different flyer hosting paths (`/volantini/cia/` vs `/volantini/dao/`).
- Tuday Conad (small convenience, ~210 sqm) never publishes store-specific flyers — the `volantiniCount: 0` in the API response is the reliable signal.
- Regional flyers (e.g. `CONAD_VENETO`) are shared across all stores in a region — filtering must allow these while blocking flyers from other cities.
- The PRODOTTI ALLA CARTA loyalty flyer omits `€` after prices and uses bare `XX%` for discounts — very different from the main weekly flyer format.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| Store locator search input timeout | Playwright found the header search bar (hidden) instead of the store locator input | Changed approach: navigate with URL params pre-filled, then call API directly via `page.evaluate()` |
| Regional flyers (CONAD_VENETO) filtered as wrong-store | City filter rejected any URL not containing target city name | Replaced with `isFlierForDifferentCity()` — only rejects URLs with a known different city |
| "convenienti sempre" flyer filtered | Same city filter issue — no city in filename | Fixed by the same `isFlierForDifferentCity()` approach |
| Loyalty flyer returns 0 products | Parser anchors look for `X,YY€` but loyalty flyer uses bare `X,YY` | Added Pass 6 (bare price + unit price) and Pass 7 (standalone percentage) |

## Final State
Complete automated Conad pipeline:
- `listConadStores()` → 6 VR-province stores via API (auto-discovery, no hardcoding)
- `discoverConadFlyers(storeUrl)` → finds PDF URLs for each store via Playwright
- `parseConadFlyerText(text)` → 134 products from weekly flyer, 5 from loyalty flyer
- Runner pipeline handles the full flow: API store listing → filter stores with flyers → discover PDFs → download → parse → ingest

CLI tools:
```bash
npx tsx src/parsers/conad-flyer-discovery.ts stores     # list all VR stores
npx tsx src/parsers/conad-flyer-discovery.ts discover    # discover flyers for all stores
```

## Open Questions
- The two Verona Conad stores (Fossetto, Pisano) share the same regional flyers — the pipeline will ingest duplicates. Should deduplication happen at the PDF URL level or at the product level during ingest?
- The "convenienti sempre" PDF is a national/regional everyday low prices flyer — is it worth parsing separately or does it overlap with the existing "Bassi e Fissi" web page scraping?
- The loyalty flyer parser only catches ~5 of ~15 products due to bare price format without unit prices. Would vision/OCR on the PDF pages improve coverage for these design-heavy cosmetics flyers?
