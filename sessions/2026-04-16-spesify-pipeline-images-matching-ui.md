---
title: "SpesaBot/Spesify: images pipeline, Migross chain, canonical matching, UI branding"
date: "2026-04-16"
author: "kos-domus"
status: "ready"
tags: ["automation", "api", "scheduling", "configuration", "setup", "debugging", "performance"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-6"
---

## Objective
Major SpesaBot (Spesify) session: add product images across all chains, integrate a new supermarket chain (Migross), implement canonical product matching for cross-chain price comparison, fix pipeline bugs, deploy services, and brand the Telegram Mini App UI.

## Context
SpesaBot is an ETL pipeline that scrapes supermarket offers from 8 Italian chains in Verona province, stores them in PostgreSQL, and serves them via a Fastify API + Telegram Mini App + grammY bot. Starting state: 4 of 8 chains had product images, no canonical product matching existed, Lidl/Aldi had stale data, and the Mini App showed emoji placeholders instead of product photos.

## Steps Taken

### 1. Image extraction — Conad bassi-e-fissi
Added `image_url` field to `ConadProduct` interface and updated the parser regex to capture `![alt](url)` markdown tags that were already present in the deep-research pipeline output.
- Parser: `src/parsers/conad.ts` — added optional capture group `(?:\s*!\[[^\]]*\]\((?<imageUrl>[^)]+)\))?` to the product regex
- Tested against real data: 507/511 products got images (99.2%)
**Result**: Conad bassi-e-fissi now has 99% image coverage

### 2. Image enrichment — Eurospin via digitalflyer API
Discovered that the Eurospin SMT Digital Flyer API at `digitalflyer.eurospin.it` returns fully structured product data including per-product image URLs.
- Created `fetchEurospinProductImages()` in `src/parsers/eurospin-url-discovery.ts`
- Added `productImages` field to `JobResult` interface in `ingest.ts`
- Enrichment logic matches PDF-parsed products to API products by normalized name (91% match rate)
- Matching handles slash variants and substring matches
**Result**: 148/162 Eurospin products enriched with images

### 3. Despar & Conad Flyer — dead ends
- **Despar**: iPaper platform only serves full-page scan images, no per-product images. No enrichment API available.
- **Conad Flyer**: PDF-based, images require EAN barcodes which aren't in the text. No product search API on conad.it.
**Result**: Documented as "no viable image source"

### 4. New chain: Migross
Migross uses the same SMT Digital Flyer platform as Eurospin. Built a pure API integration:
- Created `src/parsers/migross.ts` — fetches all products across ALL active promotions (9 promos), deduplicates by name+price
- API flow: `POST /digitalflyer/oauth/token` → `GET /stores/nazionale/promotions` → `GET /promotions/{alias}/stores/nazionale/products`
- Client auth: `Basic M2JhMWQx...` (public SPA credentials from Nuxt.js bundle)
- Added Migross handler in `runner.ts` — creates synthetic JobResult with products as `extracted` array
- DB migration `005_add_migross.sql`, 26 VR-province stores inserted
- **956 unique products, 99.4% with images, 157 distinct brands**
**Result**: Migross fully integrated and automated

### 5. Tosano research — blocked
- `latuaspesa.com` (previous e-commerce API) is down/unreachable
- `shop.supertosano.com` requires login (ExtJS app)
- `supertosano.com` is a static corporate site with no offers page
- PromoQui/DoveConviene have flyer listings but no structured product data
**Result**: Parked until data source becomes available

### 6. Lidl aggregation bug fix
**Problem**: Lidl has 15 weekly category URLs. Each was ingested separately via `ingestJobResult()`, which cleaned and re-inserted the same campaign each time — only the last page's products survived (7 out of 143).
**Fix**: Modified `runner.ts` to detect national chains with multiple URLs (`isNational`), aggregate all page results into a single synthetic JobResult, and ingest once.
- Added `readFileSync` import, `aggregatedResults` array, batch ingestion after scraping
**Result**: Lidl went from 7 to 140+ active offers

### 7. Lidl image extraction
The Lidl parser regex already had an optional image capture group, but it looked for images BEFORE the product link. In the actual markdown, images appear AFTER as `[![alt](imgUrl)](href)`.
- Updated `src/parsers/lidl.ts` to also check for linked images after the product link
- Filters to only accept `lidl.it/assets/gcp*` URLs (skip badge/seal icons)
**Result**: 100% image coverage across all 15 Lidl category pages

### 8. Canonical product matching
Built `src/canonical-match.ts` — rule-based matching:
- Normalizes product names: strips brand, removes Italian articles, normalizes quantity formats, strips accents
- Builds match key: `brand|normalized_name|normalized_quantity`
- Groups SKUs by match key, creates canonical `products` entries, links via `product_id`
- Category inference from product name patterns (latticini, pasta, bevande, etc.)
**Result**: 14,079 SKUs → 4,076 canonical products, 100% matched

### 9. LLM-assisted matching
Built `src/llm-match.ts` — finds candidate pairs (same brand, different chains, trigram similarity > 0.3), asks LLM to confirm matches.
- Provider chain: Z.AI (glm-5.1) → GPT-5.4 (OpenAI) → Gemini Flash
- Batched in groups of 30 pairs to avoid timeouts
- Found 6 real matches: Santa Lucia Mozzarella, Lavazza Qualità Rossa, Barilla Integrale Mezze Penne, Pan e Cioc, Tartelle Cuor di Mela, Cracker Farina Sostenibile
**Result**: Cross-chain products: 36 → 42

### 10. Price comparison views
Created `db/006_price_comparison_views.sql`:
- `cross_chain_prices` — all active offers grouped by canonical product
- `best_prices` — cheapest chain per product
- `price_spread` — products at 2+ chains with savings percentage
- `category_price_leaders` — cheapest chain per category
Added API endpoints: `/api/price-spread`, `/api/product/:id/prices`
**Result**: 23 products with active cross-chain comparison, up to 38% savings visible

### 11. Famila discount bug fix
**Problem**: Some Famila products showed -100% discount. Root cause: loyalty point values (e.g., "100 punti fidelity") were being parsed as discount percentages via the `parseMechanic` fallback.
**Fix**: Added sanitization in `ingest.ts` — discounts >= 90% are discarded as data errors. Fixed 19 existing rows.

### 12. Systemd services
Created always-on services for API and bot:
- `~/.config/systemd/user/spesabot-api.service` — Fastify on port 3080
- `~/.config/systemd/user/spesabot-bot.service` — grammY long-polling
- `~/.config/systemd/user/spesabot-matching.timer` — Tue/Thu/Sat 05:00
- All source secrets from `~/.openclaw/op-env-cached.sh` via `set -a; source ...; set +a`
**Gotcha**: DB password was `spesabot_local` not `spesabot` — had to `ALTER USER` to sync.

### 13. Store data
- Inserted 26 Migross VR-province stores (Superstore, Supermarket, Market, Cash&Carry) from the API
- Inserted 7 Rossetto stores from `stores.yaml` config
- Fixed `/api/stores` to remove `WHERE s.location IS NOT NULL` filter
- Total stores: 130 (was 97)

### 14. Store names on product cards
**Problem**: Famila products didn't show which store had the offer; national chains showed nothing.
**Fix**: Added `LEFT JOIN stores` + `s.name as negozio, s.city as citta` to all API endpoints (search, deals, categories, chain). Updated webapp `productCard()` to render store info. National chains show "Tutti i punti vendita".

### 15. Webapp images
**Problem**: Product cards showed emoji placeholders instead of actual product photos.
**Fix**: Updated `productCard()` in `dist/webapp/app.js` to render `<img>` with lazy loading and emoji fallback on error.

### 16. Spesify UI branding
Applied brand identity from the Spesify logo (teal cart + coral pin):
- Rewrote `dist/webapp/style.css` with brand palette
- **Critical learning**: Telegram WebView injects `--tg-theme-*` CSS variables that override `var()` fallbacks. Must use `!important` on all brand colors.
- Colors: coral background (#fdf0ed), teal text (#2d6b6b), white cards, coral prices (#e8735a), green discounts (#2d8b5a)
- Cache busting: `?v=N` query string on CSS/JS references in `index.html`

## Configuration Changes
- `configs/stores.yaml` — added `migross` chain config (strategy: api, source_type: smt_digital_flyer)
- `~/.config/systemd/user/spesabot-api.service` — new (always-on API)
- `~/.config/systemd/user/spesabot-bot.service` — new (always-on bot)
- `~/.config/systemd/user/spesabot-matching.service` — new (canonical + LLM matching)
- `~/.config/systemd/user/spesabot-matching.timer` — new (Tue/Thu/Sat 05:00)
- DB: `005_add_migross.sql`, `006_price_comparison_views.sql`, stores for Migross + Rossetto, view grants to `spesabot` user

## Key Discoveries
- **Eurospin API goldmine**: The SMT Digital Flyer API returns fully structured product data (brand, EAN, prices, category, images) — could replace the PDF parser entirely
- **Migross uses the same SMT platform** as Eurospin — same API patterns, different credentials
- **iPaper (Despar) has no per-product images** — only full-page scans, no enrichment API
- **Lidl images appear AFTER the product link** as `[![alt](img)](href)`, not before as the parser assumed
- **National chain aggregation bug**: ingesting multi-URL chains one page at a time causes campaign cleanup to wipe previous pages
- **Telegram WebView CSS override**: `var(--tg-theme-*, fallback)` fallbacks don't work in dark mode — the Telegram vars ARE set, just to dark values. Must use `!important`.
- **Z.AI rate limiting**: glm-5.1 returns 429/500 under batch load. GPT-5.4 and Gemini Flash are more reliable for batch operations.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| Lidl only 7 active offers | Each page ingestion cleaned the same campaign | Aggregate all pages into single JobResult for national chains |
| Famila -100% discount | Loyalty points "100" parsed as discount % | Discard discounts >= 90% as data errors |
| `EADDRINUSE: 3080` | Stale API process from manual test | `fuser -k 3080/tcp` before service start |
| `password authentication failed` | Service had wrong DB password | Synced with `op-env-cached.sh`, removed hardcoded `DATABASE_URL` from service |
| `permission denied for view price_spread` | Views created as postgres, queried as spesabot | `GRANT SELECT ON view TO spesabot` |
| Webapp images not showing | `productCard()` rendered emoji, ignored `image_url` | Added `<img>` tag with fallback |
| Dark search bar in Mini App | Telegram WebView dark theme CSS vars | Forced all colors with `!important` |
| Z.AI timeout/429 on batch matching | API rate limiting under load | Added GPT-5.4 and Gemini fallback chain |

## Final State
- **8 active chains**: Migross, Famila, Conad, Eurospin, Despar, Lidl, Aldi, Rossetto
- **3,865 active offers**, 90% with product images
- **4,076 canonical products**, 42 cross-chain comparable
- **4 systemd services** running (API, bot, pipeline timer, matching timer)
- **Spesify-branded Telegram Mini App** live at app.spesify.xyz
- **Price comparison** working with up to 38% savings detected

## Open Questions
- **Tosano**: When will `latuaspesa.com` come back online?
- **Cross-chain matching coverage**: Currently 42 products — will grow as more weekly data flows in. LLM matching at lower similarity thresholds (0.2-0.3) could find 300+ more candidates but with high false-positive risk.
- **Quantity normalization**: Some cross-chain price comparisons are invalid due to different package sizes (e.g., Formaggio Bastardo €1.29/piece vs €11.90/kg)
- **Confronta Prezzi tab**: Mini App doesn't yet have a dedicated price comparison screen
