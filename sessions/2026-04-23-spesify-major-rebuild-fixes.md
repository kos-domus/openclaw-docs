---
title: "Spesify: full audit fixes, UI recovery, per-store Migross, 4 bug fixes"
date: "2026-04-23"
author: "kos-domus"
status: "processed"
tags: ["spesabot", "spesify", "api", "webapp", "ingest", "migross", "postgis", "security", "ui"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-7 (1M context)"
---

## Objective

Very long working session covering: executing the entire code-audit punch list
(9 findings across security, ingest, schema, analytics), restoring the
sophisticated Spesify Mini App UI after a deploy mishap wiped it, and fixing a
second wave of 4 UX bugs found while using the app.

Main outcomes:

- DB is now consistent — 0 expired offers, 0 orphans, automated cleanup
- Pipeline is genuinely automated end-to-end (publication-id discovery,
  validity date extraction, per-page sub-promo detection, retries, cost-aware
  chain ordering, OOM headroom)
- Spesify Mini App sidebar+home UI restored and **committed to git** (it was
  ephemeral in `dist/` before and almost lost forever)
- Migross offers now attribute to specific stores via format-aware fan-out
- EAN, specifications, promotion analytics views added
- Gruppo Poli seeded (Poli, Orvea, Regina, Amort) as placeholders

## Context

Starting state for the day:

- ~2833 expired offers still cluttering `offers`
- Pipeline OOM-killed on Apr 21 (MemoryMax=2G), leaving despar/aldi/pam/
  rossetto stale for 7-12 days
- Publication-id for Shopfully chains (MD, CRAI, DPIU, Esselunga) still
  required manual weekly update
- A comprehensive audit document had been written
  (`SPESABOT_AUDIT_PARSING_SICUREZZA_EFFICIENZA_2026-04-22.md`) with 9 findings
- Migross offers all stored `store_id NULL` (chain-wide), so UI couldn't tell
  users which Migross store carried which offer

Mid-session mistake: while deploying a 2-line fix to `src/webapp/app.js`, a
`cp -r src/webapp/* dist/webapp/` overwrote the `dist/webapp/` that had been
the source of truth for the full Spesify UI (sidebar, home screen, profile,
chain-info, nearby, watches, etc.). That UI had never been committed to git
— `dist/` is gitignored. Recovery required reading the new UI description
from screenshots the user sent and rebuilding from the `style.css` class set
that survived.

## Steps Taken

### 1. Audit fixes (9 findings)

1. **Critical auth bypass** on profile/preferences/watchlist endpoints.
   Removed all `try { validateTelegramAuth(req, {} as any); } catch {}`
   fallbacks that allowed clients to pass arbitrary `telegram_user_id` in
   query/body. Replaced with `preHandler: validateTelegramAuth` where the
   endpoint requires auth, or a new `tryValidateTelegramAuth(req)` helper
   (non-throwing) for read endpoints that should degrade gracefully.
2. **Ingest non-atomic** — `cleanCampaignOffers(id)` used to run outside the
   transaction. Moved the DELETE inside `BEGIN/COMMIT` so a mid-ingest crash
   no longer leaves the campaign empty.
3. **conad-flyer store model** — added optional `store` field to
   `ingestFlyerText`, runtime store resolution in `ingest.ts`, and scoped the
   clean step to `(campaign_id, store_id)` so multiple Conad stores in the
   same week don't wipe each other.
4. **user_profiles schema drift** — migration
   `009_user_profile_auth_columns.sql` adds `username`, `display_name`,
   `onboarded` columns + partial unique indexes (email, username).
5. **Chain registry/seed misalignment** — migration
   `010_seed_additional_chains.sql` seeds md/crai/dpiu/esselunga;
   `ChainStrategy` type extended to include `'api'`.
6. **Canonical matching not in flow** — `scripts/run-pipeline.sh` now runs
   `dist/canonical-match.js` inline after ingest; LLM pass stays on its own
   schedule.
7. **refresh_product_tags full rewrite** — migration
   `011_refresh_tags_incremental.sql` + `013_fix_refresh_tags_overload.sql`
   makes it incremental (`since_hours` parameter, default 6).
8. Validity dates already addressed via Shopfully vision extraction.
9. Shopfully vision cost/fragility addressed via retry + page-discovery +
   sweep.

### 2. Pipeline reliability

- **Publication-id auto-discovery** (`src/parsers/shopfully-pub-discovery.ts`)
  — scrapes doveconviene.it for the current Shopfully pub id, falls back to
  env var if discovery fails. Logs
  `publication id rotated: X → Y` when a new flyer cycle starts.
- **Flyer header date extraction** — Gemini vision call on page 1 reads
  "PROMOZIONI VALIDE dal X al Y" and uses that as the campaign window.
- **Per-page sub-promo detection** — second Gemini vision call per page
  detects narrower banners (e.g. "Weekend più uno 01-02-03 MAGGIO + 04
  MAGGIO") and overrides `valid_from`/`valid_to` for products on that page.
- **Retry on transient errors** — `callGeminiOnPage` retries once on 429/5xx/
  timeout. Cuts MD vision error rate from 35% → 5% on good runs.
- **Page sweep** — full-page vision call per page recovers products missed
  by Shopfully's hotspot metadata (e.g. "Sapori dalla Toscana" section on MD
  p15/p16 only had 6 hotspots for 9 visible products). Dedup by
  `(page, quantity, price)` so sweep products don't collide with hotspots
  even when naming differs.
- **Cost-aware chain ordering** (`src/store-registry.ts`) — listChains()
  returns chains ranked by resource cost: API (conad, migross) → HTML
  (aldi, lidl, rossetto, despar, pam, martinelli) → Shopfully vision
  (md, crai, dpiu, esselunga) → Playwright (famila, conad-flyer). OOM
  failures now only affect cheap-chain leftovers.
- **MemoryMax=3.5G / MemoryHigh=3G** in the systemd service (was 2G/1.5G).

### 3. DB cleanup + automation

- **Cleanup functions** — migration `012_cleanup_expired_offers.sql` adds
  `cleanup_expired_offers(grace_days)` and
  `cleanup_abandoned_campaigns(grace_days)`.
  `scripts/run-pipeline.sh` calls both with grace=2 after each ingest.
- **One-time manual cleanup** deleted 2,832 expired offers + 12 orphan
  campaigns + 1,237 orphan SKUs. DB is now integrity-clean.

### 4. EAN + specifications + promotion analytics

Migration `015_ean_specs_and_analytics.sql`:

- `product_skus.ean TEXT` with partial unique index on `(chain_id, ean)` +
  global lookup index.
- `product_skus.specifications JSONB` (free-form: energy class, alcohol %,
  category, dimensions, etc.). COALESCE-merge on conflict so progressive
  enrichment works.
- `backfill_missing_images()` function — fills NULL image_url from canonical
  siblings. Invoked inline from `run-pipeline.sh` after canonical matching.
- `v_promotion_analytics` view — per chain/week/category/mechanic counts,
  avg discount, price min/max. Uses LATERAL unnest of tags so categories
  explode correctly.
- `v_current_promotions` view — snapshot of active offers per chain.

Ingest + parsers updated:

- `src/ingest.ts` reads `product.ean`/`gtin`/`barcode` (digit-only length
  gate 8–14) and `product.specifications` JSONB.
- Shopfully vision prompt extended: `"ean"` + `"specifiche"` fields.
- Migross parser attaches EAN from `item.code.value` (when 8–14 digits) +
  specifications from DIMENSION/MEASURE-UNIT/CATEGORY/initial KG price.

### 5. Gruppo Poli scaffolding

Migration `014_seed_gruppo_poli.sql` adds 4 new chain rows (poli, orvea,
regina, amort). `configs/stores.yaml` has entries with
`strategy: api, parser: unimplemented` so the runner skips them cleanly
until a bespoke scraper is built. The aggregator pages on
doveconviene.it/promoqui.it point to a DIFFERENT "Poli" (Umbrian) — the
real Gruppo Poli (Trentino-Veneto) hosts flyers behind an ASP.NET
ViewState form on gruppopoli.it, no clean API.

### 6. UI mishap + sidebar restore

The `cp -r src/webapp/* dist/webapp/` command ran as part of deploying
loyalty-flag fixes overwrote the full sidebar+home UI (13KB `index.html`,
52KB `app.js`) that lived only in `dist/`. Recovery path:

- Tried git reflog, git stash, VS Code file history, cloudflared cache,
  `~/.cache/` — nothing.
- Found Claude Code session transcripts at
  `~/.claude/projects/-home-kos-job-desk/*.jsonl` containing partial Read
  captures. Got the recovered 6-tab `index.html` fully (clean) but the
  sidebar-version `app.js` only as fragments (81% coverage, with duplicate
  function definitions from overlapping Edit operations).
- User sent screenshots of the real UI: teal `#topbar`, slide-in `#sidebar`
  with 12 items in 3 groups (5 main + 6 services + 1 settings), branded
  home screen with 4 quick-action tiles + live stats.
- Rebuilt both files from scratch using the screenshots + the CSS selectors
  that survived (`.home-brand`, `.home-action`, `.sidebar-item`,
  `.chain-info-grid`, `.profile-card`, etc.) + the detailed session note
  from Apr 17.

**Committed the rebuilt UI to git** (`src/webapp/*` is now the source of
truth) + added a `build` script (`tsc && cp -r src/webapp dist/`) so the
UI can never be ephemeral again. Plus a `.webapp-backups/` folder for
belt-and-braces snapshots.

### 7. Second wave of 4 UI bugs (after restore)

1. **"Vicino a me" only found Famila San Pietro** — was using
   `/api/offers/nearby?limit=1` + city-substring matching. Fixed by
   adding `GET /api/stores/nearby?lat&lng&radius_km=20` (PostGIS
   `ST_DWithin`) that returns all stores in range; `selectNearbyStores()`
   now toggles every matching store. Result: 85 stores across 7 chains.
2. **Profile screen blank** — `/api/profile` required HMAC auth and
   returned 401 when initData was missing/expired, which the UI couldn't
   render. Changed to return `{authenticated, registered}` at 200 OK;
   register endpoint still requires HMAC so security is unchanged. UI now
   has 3 clean states (registered card / register form / "apri da Telegram"
   hint).
3. **Info Catene too short** — expanded `CHAIN_INFO` from 8 stubs of ~3
   tips each to 11 full cards (Famila, Conad, Lidl, Eurospin, Despar, MD,
   Aldi, Rossetto, Migross, CRAI, DPiù) with blurb + 4-6 tips + "good to
   know" footer. Rendering adds a paragraph + italic footer.
4. **Watch alerts couldn't set max_price** — `window.prompt` is blocked in
   Telegram Mini Apps. Replaced with inline register-form style panel with
   `<input type="number" inputmode="decimal">` for max price.

### 8. Migross per-store fan-out

The user's final complaint: "Migross offers are different based on the store
you're in, and the app doesn't tell me which stores".

Root cause: Migross SMT API runs 15+ parallel promotions (Superstore,
Market&Supermercati, Catalogo Vini, Grandi Marche, Beautypharma, Promozione
Retail, Pet Store Buddy, etc.). We were fetching each with
`/promotions/{alias}/stores/nazionale/products`, deduping by name+price,
and writing one `store_id NULL` offer per product.

Fix:

- `MigrossProduct` gained `promotion_description` + `promotion_alias`
  fields. Plumbed through `parseApiProduct` + `fetchPromoProducts`.
- Dedup changed from Set to Map: when the same (name, price) is seen in
  multiple promotions, the descriptions are **merged**
  ("Superstore + Market&Supermercati") so ingest can fan out to the union
  of applicable stores.
- New `migrossFormatsForPromotion(description)` helper classifies by
  keywords: "Superstore" → supermarket, "Market" → market,
  "Cash&Carry" → cashcarry, "Pet Store Buddy" → skip (not grocery),
  generic → all.
- Ingest now preloads Migross stores indexed by format (by name prefix:
  "Migross Supermarket *" / "Migross Market *" / "Migross Cash&Carry"),
  then for each Migross product writes ONE offer row per applicable store
  instead of a single chain-wide row.
- Clean step updated to wipe all Migross offers (chain-wide + per-store)
  on each run.

Result: 1,120 chain-wide offers → **25,368 per-store offers** across 1,106
distinct SKUs. `/api/deals/top` still groups by (sku, chain) so UI sees
one card per product per chain, but the `stores[]` list now names every
branch that carries it.

## Configuration Changes

New DB migrations (all idempotent):

- `009_user_profile_auth_columns.sql` — username/display_name/onboarded + uniq indexes
- `010_seed_additional_chains.sql` — md/crai/dpiu/esselunga seed
- `011_refresh_tags_incremental.sql` — incremental refresh_product_tags
- `012_cleanup_expired_offers.sql` — cleanup_expired_offers + cleanup_abandoned_campaigns
- `013_fix_refresh_tags_overload.sql` — fix refresh_product_tags overload ambiguity
- `014_seed_gruppo_poli.sql` — Gruppo Poli (poli/orvea/regina/amort)
- `015_ean_specs_and_analytics.sql` — ean/specifications columns + analytics views

Pipeline script (`scripts/run-pipeline.sh`) now runs:

1. `dist/runner.js` (fetch + ingest all chains in cost-first order)
2. `dist/canonical-match.js` (rule-based cross-chain matching, best-effort)
3. `backfill_missing_images()` SQL function
4. `cleanup_expired_offers(2)` + `cleanup_abandoned_campaigns(2)` (grace=2 days)
5. `dist/notify-watches.js`

Systemd service: `MemoryMax=3.5G`, `MemoryHigh=3G` (was 2G/1.5G).

`package.json` scripts:

- `build`: `tsc && cp -r src/webapp dist/`
- `deploy-webapp`: `cp -r src/webapp dist/ && echo 'webapp deployed to dist/'`

## Key Discoveries

- **Claude Code session transcripts are a disaster-recovery goldmine.** The
  JSONL files at `~/.claude/projects/-home-kos-job-desk/*.jsonl` preserve
  every Read/Write/Edit operation. When the Spesify UI was lost, stitching
  together Read captures at different offsets recovered the clean 6-tab
  `index.html` 100% and ~81% of a historical `app.js`. Not enough to
  restore the sidebar version (which was built via 145 incremental Edits
  without any full-file Read captures), but enough to bootstrap a rebuild.
- **`dist/` gitignored + UI lived only in `dist/`** is a catastrophic
  anti-pattern. Never let generated output be the source of truth. Fix:
  keep `src/webapp/` committed and have `npm run build` copy to `dist/`.
- **PostgreSQL regex word boundary is `\y`, not `\b`** (learned Apr 17,
  re-encountered).
- **PostgreSQL `COUNT(...)::int FILTER (...)` is a syntax error** — must
  be `(COUNT(...) FILTER (...))::int`. Surfaced as HTTP 500 on /api/deals.
- **CREATE OR REPLACE FUNCTION with a new parameter signature creates an
  overload, not a replacement.** `refresh_product_tags()` +
  `refresh_product_tags(since_hours INT)` both existed simultaneously,
  making `SELECT refresh_product_tags()` ambiguous. Fix: DROP both
  signatures first, then CREATE.
- **Migross SMT API doesn't expose per-store offer visibility directly.**
  The promotion description is the only signal — "Superstore" vs
  "Market&Supermercati" — and the mapping to actual stores happens via
  name prefix matching in our DB.
- **Gemini 2.5 Flash 503 outages are frequent.** A single bad API day
  dropped product extraction from 249 to 59 (75% failure). Retry-once
  cut the impact; true stability would need a secondary model or
  queue-and-retry pipeline.
- **Telegram Mini App webview caches aggressively** even with
  `Cache-Control: no-cache`. Version-bumping query strings (`?v=TS`) on
  each JS/CSS fetch is the only reliable way to force re-fetch.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `function refresh_product_tags() is not unique` | Migration 011 created a 2nd overload of existing function | Migration 013 drops both signatures then recreates with default param |
| `syntax error at or near "FILTER"` on /api/deals/top | `COUNT(...)::int FILTER (...)` wrong order | `(COUNT(...) FILTER (...))::int` |
| Famila ingest OOM killed the pipeline mid-run | `MemoryMax=2G` + Famila Playwright-heavy + Famila ran first alphabetically | Raise limits to 3.5G + reorder chains by cost |
| Mini App main page disappeared | `cp src/webapp/* dist/webapp/` overwrote sidebar UI that lived only in `dist/` | Recover from session-transcript Read captures + rebuild from screenshots, commit `src/webapp/` to git |
| "Vicino a me" found only 1 store | Was using `/api/offers/nearby?limit=1` + fuzzy city-substring matching | New `/api/stores/nearby?radius_km=20` with PostGIS ST_DWithin |
| Profile screen blank | `/api/profile` returned 401 when HMAC missing; UI couldn't render that | Endpoint now returns `{authenticated:false, registered:false}` at 200 OK |
| User couldn't set max price on watch alerts | Telegram Mini Apps block `window.prompt` | Inline register-form style panel with `<input type="number">` |
| Migross offers said "available at Migross" without stores | All offers stored `store_id NULL` (chain-wide) | Fan out per-store using promotion→format mapping |
| `NET::ERR_CERT_COMMON_NAME_INVALID` on t.me for a friend | Environmental (captive portal / MITM / ISP DNS hijack) | Not our bug — user's friend's network issue |

## Final State

### DB

- 10 chains actively scraping (aldi, conad, crai, despar, dpiu, eurospin,
  famila, lidl, md, migross, rossetto)
- 6 chains scaffolded but no scraper yet (poli, orvea, regina, amort, pam,
  martinelli)
- ~18k active offers + Migross fanned out to ~25k per-store rows = ~40k
  total active offers
- 16k+ SKUs, 100% matched to canonical products
- 0 expired, 0 orphan campaigns, 0 orphan SKUs

### Spesify Mini App (live at https://app.spesify.xyz/webapp/)

- Teal topbar + slide-in sidebar with 12 items in 3 groups (5 main, 6
  services, 1 settings)
- Branded home screen: wordmark + coral tagline + active-chain list + 4
  quick-action tiles + live stats from /api/status
- 12 screens: Home, Cerca, Offerte, Categorie, Info Catene, Vicino a me,
  I miei avvisi, Negozi, Carte Fedelta, Profilo, Analytics (owner-only),
  Preferenze
- All screens wired to the API
- Coverage chips on product cards (📍 N negozi / 🌐 Rete nazionale)
- National chains visible in Negozi + Preferenze
- Inline form for add-watch with numeric max_price
- Chain Info with 11 richly-documented cards

### Pipeline

- Tue + Fri 03:00 CEST via systemd timer, `Persistent=true`
- Cost-first chain ordering, OOM-protected
- Auto-discovers Shopfully pub ids from doveconviene.it
- Extracts flyer header dates + per-page sub-promos
- Retries transient Gemini errors
- Page-sweep recovery for Shopfully hotspot gaps
- Inline canonical matching + image backfill + expired cleanup
- Watchlist notifications at end

### Git state

Main branch, all committed:

- `6000b4c` Restore full Spesify Mini App UI (sidebar + branded home)
- `ab8856d` Fix 4 UI bugs: nearby-stores, profile, chain-info, watch form
- `37bd477` Expand Migross offers per-store using promotion→format mapping
- `4d55d0f` Bump webapp cache-bust version to timestamp

`src/webapp/` is now the source of truth. `dist/webapp/` is a build output.

## Open Questions

- **Gruppo Poli scraper** — ASPX ViewState form is the realistic path; needs
  Playwright session with the store-picker form. Deferred.
- **Analytics dashboard UI** — `/api/analytics/summary` endpoint exists
  (owner-only) but the webapp screen just dumps JSON. Should get proper
  charts/cards.
- **"Sapori dalla Toscana" hotspot gaps** — Shopfully metadata only covers
  ~6 of 9 visible products on MD p15/p16. The page sweep recovers most but
  not all. Would need to split the hotspot region geometrically.
- **Esselunga empty hotspots** — Shopfully API returned 0 hotspots on the
  current Esselunga publication. Could be a one-off or the publication ID
  is stale. Worth re-checking next Fri pipeline run.
- **Telegram cache stickiness** — even with Cache-Control + `?v=TS` bumps,
  the iOS Telegram WebView can serve stale content for hours. No clean
  solution; users need to clear app cache manually.

---

*Status: processed by kos-docs elaboration.*
