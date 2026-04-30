---
title: "Fleet morning fixes (Telegram, Kai briefing, daily memory, MC Todo, staggering) + SpesaBot consolidation, Esselunga per-store fan-out, expiry feature, cross-chain image registry"
date: "2026-04-30"
author: "kos-domus"
status: "ready"
tags: ["cron", "scheduling", "automation", "multi-agent", "debugging", "troubleshooting", "memory", "configuration", "git"]
openclaw_version: "2026.4.8"
environment:
  os: "Linux 6.17.0-22-generic"
  ide: "VSCode native extension"
  model: "claude-opus-4-7[1m]"
---

## Objective

Two distinct workstreams in one session:

1. **Fleet morning audit** — verify OpenClaw agents (Kai/Kos/MC) are operational, fix any visible issues, then add a new feature (MC Morning Todo List that synthesizes a daily action list from Obsidian + specialist reports).
2. **SpesaBot consolidation** — bring 7 days of uncommitted work into git, fix broken Esselunga ingest (only 24 stale offers in DB from a dead pubid), add a per-store fan-out pattern, ship an expiry-tracking feature, then build a persistent cross-chain image library.

## Context

- **Fleet state at start**: gateway up 1d 13h, 8 agents registered, ~50 cron jobs, but yesterday's MC Daily Briefing failed delivery (Telegram "chat not found"). Kai briefing had a recurring `tmp-reminders.json` ENOENT. Daily memory files for cos/family generate ENOENT noise at every session reset.
- **SpesaBot state at start**: 21 files modified (3104 insertions / 130 deletions) since last commit on Apr 23, plus 14 untracked DB migrations (003-016), plus 5 untracked critical source files (canonical-match.ts, crypto.ts, llm-match.ts, notify-watches.ts, notify-grocery-list.ts). Production was running on uncommitted code. Esselunga had only 24 offers in DB (a stale "Bottega" catalog), valid_to expired 4 days ago.

## Steps Taken

### 1. Fleet status check
Read `~/Obsidian-Personal/Daily/2026-04-29.md`, listed all cron jobs with `openclaw cron status`, surveyed agent workspaces and cron run history.

**Result**: identified 4 issues:
- ⚠ MC Telegram delivery to Job-desk group failing with "chat not found"
- ⚠ Kai morning briefing — `tmp-reminders.json` ENOENT
- ⚠ Daily memory `<workspace>/memory/<today>.md` ENOENT at session reset (recurring noise)
- ⚠ Skill GWS sought in `/root/...` with EACCES

### 2. Fix #1 — MC Telegram delivery (already self-fixed)
Investigated by reading the runtime delivery code in `~/.npm-global/lib/node_modules/openclaw/dist/agent-delivery-DLNPPuOh.js` and `extensions/telegram/send-B5NZ9QZr.js`. Verified bot tokens via Telegram getMe API:
- Default bot `Kos_OC_bot` is **not** a member of Job-desk group → "chat not found"
- Work bot `Workspace00_bot` is a member ✓
- The MC Daily Briefing job's `delivery` block had `accountId: "work"` set on Apr 28 19:27 (visible in `jobs.json.before-mc-accountid-fix-2026-04-29` backup), but the failed run was on Apr 29 06:32 — earlier in the day, before the fix landed.

Verified the fix was already in place by sending a sanity message via the work bot to the group: `msg_id=110, ok=true`.

**Result**: confirmed the user's previous-evening fix was correct; no action needed beyond verification.

### 3. Fix #2 — Kai briefing reminder fetch
Trace from session JSONL: the job spawns `gws drive files get --params {fileId,alt:media} -o /tmp-reminders.json`, and `gws drive files get -h` documents `-o` as "Output file path for **binary** responses". Reproduced: for JSON (text) responses, `-o` is a no-op — content goes to stdout, file never written. The subsequent `read` tool call ENOENTs.

Fix: edit the cron job's prompt to use stdout redirect:
```bash
gws drive files get --params '...' > /home/kos/.openclaw/workspace-family/tmp-reminders.json 2>/dev/null
```

Applied via `openclaw cron edit 113b78c3-bb63-430e-9e3b-0bb66933487d --message "..."`.

**Result**: ✓ next 06:50 run will write the file properly.

### 4. Fix #3 — Daily memory ENOENT noise
Located the source in OpenClaw runtime: `dist/startup-context-T0Kl6rrH.js` `buildSessionStartupContextPrelude()` reads `memory/<stamp>.md` for each recent stamp at every new session. ENOENT logs are emitted but the function gracefully skips empty/missing files. The noise pattern correlates exactly with the daily 04:00 session reset.

Created systemd user timer pattern (consistent with existing spesabot timers):
- `~/.openclaw/touch-daily-memory.sh` — script that creates empty files via `: > "$file"` for cos and family workspaces (runtime skips empty files via `if (!content?.trim()) continue;`)
- `~/.config/systemd/user/openclaw-daily-memory-touch.service` — Type=oneshot, runs the script
- `~/.config/systemd/user/openclaw-daily-memory-touch.timer` — `OnCalendar=*-*-* 03:55:00, Persistent=true`

Enabled and ran once to pre-create today's files.

**Result**: timer scheduled daily at 03:55 (5min before session reset). ENOENT noise gone.

### 5. Fix #4 — New MC Morning Todo cron job
Designed a separate-job approach (Option B from a 3-option design exchange): runs at 07:10 (after MC Daily Briefing has written the Obsidian Daily), reads:
1. `/home/kos/Obsidian-Personal/Daily/<today>.md` (sections: Today's projects, Blocks, Risks, Wins)
2. The most recent report from each specialist Drive folder (Backend / Frontend / CSO / OrchArch — 4 calls to `gws drive files list orderBy=modifiedTime desc pageSize=1`, then `gws drive files get` per latest file)

Synthesizes a 5-10 item to-do list with format `[Pn, X] action — perché (source)`:
- Pn = P0 (block obbligatorio) | P1 (importante) | P2 (sfondo)
- X = S (≤30min) | M (1-2h) | L (≥3h)

Posts to Telegram Job-desk group via `accountId: work`.

Created via `openclaw cron add` with full prompt embedded via HEREDOC.

**Result**: ✓ job id `0b88a6d1-43fd-4f6d-abfa-53e85e914175`, first run 2026-04-30 07:00 (before staggering); after staggering moved to 07:10.

### 6. Fix #5 — Morning cron staggering
Audited the 06:00–09:00 window: 9 jobs colliding at 07:00 on Mondays (Kai briefing + Cos docs + MC Todo + family weekly briefings). Risk: token rate-limit burst, race conditions (MC Todo reading Obsidian before MC Briefing wrote it).

Re-scheduled the 3 LLM-heavy daily jobs:
- 06:30 MC Daily Briefing (unchanged — produces Obsidian Daily first)
- **06:50** Kai Daily morning briefing (was 07:00)
- **07:10** MC Morning Todo (was 07:00)
- **07:30** Cos Daily docs elaboration (was 07:00)

Applied via `openclaw cron edit <id> --cron "<expr>"` for each. Family promemoria (vetro/mensa/ritiro) left at 07:00 — they're pure-send WhatsApp, lightweight.

**Result**: ✓ 20-minute buffer between every heavy job; respects causal dependency (MC Todo needs Obsidian Daily).

### 7. SpesaBot working tree consolidation
Audit revealed: 21 modified files + 14 untracked DB migrations + 5 untracked critical source files. Production running on uncommitted code since Apr 23. Split into 9 logical commits:

1. `chore: organize audits, ignore data/, add DOSSIER` — moved 4 audit files into `audits/`, ignored 29MB `data/product-images/`, tracked DOSSIER.md
2. `db: add migrations 003-016` — 14 migrations from Apr 14 to Apr 25 (schema fixes, watches, analytics, cleanup, shopping_list)
3. `pipeline: cost-first ordering, OOM mitigation, parser fan-out, image backfill` — runner.ts + store-registry + run-pipeline.sh + populate-store-locations
4. `api: shopping list endpoints + product search`
5. `bot: refresh /start commands + Conad loyalty signup tweaks`
6. `webapp: shopping list screen + sidebar entry + cache-bust bump`
7. `parsers: Famila API, Shopfully vision, Conad-flyer Playwright, Rossetto, URL discovery` — 21 parser files (4726 lines new)
8. `core: matching engines, Telegram HMAC, watches/grocery notify, matching timer` — canonical-match.ts, llm-match.ts, crypto.ts, notify-watches.ts, notify-grocery-list.ts, scripts/run-matching.sh
9. `chore: docs + lockfile + ignore scripts/research/` — research scripts moved to ignored `scripts/research/`

Build still passes after each commit. Both spesabot-api and spesabot-bot services remained active throughout.

**Result**: ✓ working tree clean, baseline reproducible. 7 days of work recovered.

### 8. Esselunga fix — diagnose and resolve "0 offers ingested"
Investigation chain:
- DB state: 24 stale offers, valid_to 2026-04-26 (expired 4 days ago), all chain-wide.
- Discovery `resolveShopfullyPubId('esselunga')` returned `null`. Manual fetch of `https://www.doveconviene.it/volantino/esselunga` showed only city directory links (Arezzo/Siena/Perugia — 16 Centro Italia cities); no `shopfully.cloud/publication/it_it_*` embed on the national page.
- The configured pubid `813043` returned 10 valid pages with `pageEnrichments: 0` (zero hotspots).
- Vision-tested page 5 of pubid 813043: returned 11 "products" all with `prezzo_offerta: null` — these were from the **Esselunga "Bottega"** prepared-foods catalog (Pollo allo spiedo, Lasagne con ragù, Insalata Russa), not the weekly price flyer.

Root cause: TWO bugs.
1. `shopfully-vision-fetch.ts:579` had `if (hotspots.length === 0) continue;` which short-circuited the page download AND the page-sweep when a page had no hotspot metadata. Esselunga (with 0 hotspots on every page of the dead pubid) hit this branch on every page.
2. The configured pubid was a dead Bottega catalog; the real weekly pubid lives only on the city-specific page (e.g. `/verona/volantino/esselunga`), never on the national directory.

Fix:
1. Removed the early `continue` in the hotspot loop (`shopfully-vision-fetch.ts`); always download page image so the full-page sweep can run.
2. Extended `shopfully-pub-discovery.ts` to accept an optional `city` slug; `resolveShopfullyPubId` now tries the national directory first, falls back to `/{city}/volantino/{chain}` when `{SLUG}_SHOPFULLY_CITY` env var is set.
3. Updated env (`~/.openclaw/op-env-cached.sh`):
   - `ESSELUNGA_SHOPFULLY_PUBLICATION_ID`: `813043` (dead Bottega catalog) → `824458` (current weekly, discovered via Verona page)
   - `ESSELUNGA_SHOPFULLY_CITY=verona` (new)

Production test: 25 pages, 200 hotspots on 23 pages, validity 23/04→06/05, 247 products extracted (188 hotspot + 59 sweep), **246 offers ingested**. Pipeline complete in 394s.

**Result**: ✓ Esselunga went from 24 stale records → 246 fresh weekly offers. `commit 16a2e95`.

### 9. Feature: expiry badges + "In scadenza" screen
Audit: `offers.valid_to` already in DB and used in WHERE clauses, but never surfaced to UI; no sort by expiry; no dedicated screen.

Implementation:
- API: new `/api/deals/expiring?within_days=N&limit=K` (default 3, max 14, ordered by valid_to ASC then discount_pct DESC). `/api/deals/top` and `/api/offers/nearby` updated to include `valid_to` in payload.
- Webapp: `expiryBadge()` helper computes a colored chip from `valid_to`:
  - red `Scade oggi` / orange `Scade domani` / amber `Scade tra N giorni` (≤7) / neutral `Scade GG/MM` (>7)
  - Critical: parses Date in **local TZ** to avoid 1-day-early off-by-one (Postgres `date` serializes as ISO timestamp → CEST midnight UTC-shifted).
- New "In scadenza" sidebar entry between "Vicino a me" and "I miei avvisi" with sticky chip selector: Oggi / 3 giorni / 1 settimana / 2 settimane.
- Cache-bust bumped on `app.js` + `style.css`.

**Result**: ✓ `commit 7d886b8`. Smoke-test confirmed endpoint working: 5 Eurospin deals expiring with proper sort.

### 10. Esselunga per-store fan-out (after user pushed back: "le offerte sembrano essere per-store")
Investigation:
- 3 Verona stores on doveconviene anchor pages: Fiera (1020871) → pubid 824458; Corso Milano (1216011) + Fincato (560805) → pubid 824441 (Punti Fragola — fedeltà program with point-prices, NOT a weekly flyer).
- Drilled into the multiple `flyerId` values per store page: discovered hidden pubid 824460 (26 pages) for Corso Milano + Fincato.
- Rigorous comparison 824458 vs 824460:
  - All 25/26 page WEBP filenames different (physically distinct images)
  - Page 3: nearly identical content with minor OCR variations
  - Page 10: identical
  - **Page 15 completely different**: Fiera = pet food (Monge, Friskies); Corso Milano + Fincato = flowers (Phalaenopsis, Bouquet Deluxe)

Implication: chain-wide ingest would either duplicate grocery rows or surface offers in wrong stores. Needed proper per-store binding.

Implementation:
- `db/017_seed_esselunga_verona_stores.sql` — seed 3 Esselunga stores with Nominatim coords, `external_id` = doveconviene anchor id.
- `runner.ts`: new `runShopfullyMultiPub(chain, envJson, ...)` helper. Reads `{SLUG}_PUBLICATIONS` env var as JSON array of `{pubId, externalIds[]}`. Resolves external_ids → store.id once up front, then for each entry: fetch publication, attach `targetStoreIds` to synthetic JobResult, ingest.
- `ingest.ts`: respects `jobResult.results[0].targetStoreIds`, fans every offer across the listed stores. Cleanup is now scoped: `DELETE WHERE campaign_id = X AND store_id = ANY(target)` so a sibling pubid for a different store-set is not wiped on the same run.
- `server.ts`: `/api/offers/nearby` now includes `valid_to` for badge consistency.

Env addition:
```
ESSELUNGA_PUBLICATIONS='[{"pubId":"824458","externalIds":["1020871"]},{"pubId":"824460","externalIds":["1216011","560805"]}]'
```

Production runs:
- First run timed out at 20min during pubid 824460 vision phase.
- Second run (824460 only, 30min timeout): completed in 775s, 120 offers across 2 stores.

**Result**: ✓ DB final state — 87 Fiera + 120 Corso Milano + 120 Fincato = 327 store-bound rows. "Vicino a me" from Bussolengo correctly lists all 3 stores ordered by km. `commit 714fb58`.

### 11. Cross-chain product image registry
User idea: persistent memory of product images, useful when a parser can't extract one.

Audit revealed:
- `cleanup_expired_offers()` deletes only offers, never SKUs → `product_skus.image_url` is already persistent.
- Existing `backfill_missing_images()` matches only by `product_id` (canonical) — useless for chains where canonical pipeline rarely fires (Esselunga: 24/373 = 6% canonicals).
- Coverage gaps (active offers): despar 1%, aldi 10%, dpiu 26%; vs famila 99%, migross 100%.
- 30k+ existing SKUs with images can be "donated" to other chains via brand+name match.

Migration `db/018_product_image_registry.sql`:
- Table with 3 matching keys: EAN (gold) > (brand_norm, name_norm, quantity_norm) > name-only.
- `norm_text()` IMMUTABLE helper for index consistency.
- `register_product_image()` — UPSERT, EAN-priority.
- `lookup_product_image()` — STABLE, returns highest-confidence URL.
- Trigger `product_skus_register_image` AFTER INSERT/UPDATE OF image_url — auto-feeds the registry on every ingest.
- `backfill_images_from_registry()` — scans NULL image_url SKUs, applies lookup.
- One-shot seed: 32 564 existing SKU rows bootstrap the registry on migration.

`run-pipeline.sh` updated to call both backfills in sequence (canonical first, then registry).

**Result**: ✓ `commit edc781c`. Backfill healed 520 SKUs immediately. Active-offer image coverage:
- Esselunga 73% → 85%
- MD 87% → 95%
- Lidl 70% (SKUs) → 100% (active offers)
- Conad 49% → 55% (SKUs) / 90% → 99% (active)
- Despar/Aldi remain low (private-label products with no cross-chain match)

## Configuration Changes

| File | Change |
|------|--------|
| `~/.config/systemd/user/openclaw-daily-memory-touch.{service,timer}` | New systemd timer @ 03:55 daily — pre-creates empty memory files for cos/family |
| `~/.openclaw/touch-daily-memory.sh` | New script invoked by the timer |
| `~/.openclaw/cron/jobs.json` | 3 cron edits (Kai briefing prompt fix, MC Todo new job, 4 schedule changes for staggering) |
| `~/.openclaw/op-env-cached.sh` | `ESSELUNGA_SHOPFULLY_PUBLICATION_ID` updated, `ESSELUNGA_SHOPFULLY_CITY` and `ESSELUNGA_PUBLICATIONS` added |
| `~/.claude/projects/-home-kos-job-desk/memory/project_mc_morning_todo.md` | New memory entry — MC Morning Todo design + staggering rule |

## Key Discoveries

- **`gws drive files get -o <path>` is binary-only** despite no docs warning. For JSON/text responses the file is never written — output goes to stdout only. Pattern fix: `... > /path/to/file 2>/dev/null` instead of `-o`.
- **Telegram chat group migrations are silent**: the work bot can `getChat` a group, but a different bot (default account) trying to send to the same chat_id gets "chat not found". The error message points to migration/removal but the real cause was an `accountId` mismatch in the cron job's delivery config.
- **OpenClaw runtime auto-loads `memory/<today>.md` at every new session**, with `if (!content?.trim()) continue;` guard. Empty files (via `:>` or `touch`) silence the ENOENT noise without polluting the agent's startup context.
- **Cron `delivery.accountId` field is respected by the runtime** (verified by reading `jobs-Dy87ilE4.js`) but the active job had it set to `"default"` (Kos_OC_bot, not in the group) — the user had already self-fixed this 4min before the Apr 29 19:31 Evening Digest run.
- **Token rate-limit burst at 07:00**: 3 LLM-heavy daily jobs + 6 family weekly briefings (on Mondays). Spreading 3 main offenders across 06:30–07:30 with 20min buffers eliminated the contention without changing the prompts.
- **Esselunga Shopfully publication topology**:
  - National doveconviene page only embeds the viewer for some retailers (MD/CRAI/DPIU) — for others (Esselunga) the embed is on city pages.
  - One city can have **multiple pubids** for different store-groups (Verona: Fiera = 824458, Corso Milano + Fincato = 824460).
  - Different pubids share most grocery sections but diverge on verticals (page 15: pet food vs. flowers).
  - Some pubids served by doveconviene are dead catalogs (Bottega prepared-foods) with 0 hotspots and no prices — vision OCR returns `prezzo: null` on every product.
- **`product_skus.image_url` is effectively a persistent image library already** — `cleanup_expired_offers` only purges the `offers` table. The new registry just makes the lookup keys broader (EAN + brand+name+qty + name-only) than the canonical-product_id-only pattern.
- **Postgres `date` columns serialize to ISO timestamp at DB-TZ midnight** when sent over JSON. Client-side TZ parsing must use `new Date(value).getFullYear()/getMonth()/getDate()` not `slice(0,10)` — otherwise `2026-04-29T22:00:00Z` (= midnight CEST Apr 30) reads as Apr 29.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `Telegram send failed: chat not found (chat_id=-100...)` | MC Daily Briefing delivery used `accountId: "default"`; default bot (Kos_OC_bot) not in target group; work bot (Workspace00_bot) is | User had already changed `accountId` to `"work"` 4min before the Evening Digest run; verified by sending sanity message via work bot |
| `read failed: ENOENT ... /home/kos/.openclaw/workspace-family/tmp-reminders.json` | Kai briefing prompt used `gws drive files get -o`; flag is binary-only, doesn't write text responses | Edited prompt to `... > /path/to/file 2>/dev/null` |
| Recurring `read failed: ENOENT ... memory/<today>.md` for cos/family | Runtime auto-loads daily memory at every new session; files are written only by EOD wrap-up | systemd user timer `openclaw-daily-memory-touch` @ 03:55 creates empty files; runtime skips empty content silently |
| Esselunga pipeline returned 0 products despite 25 pages | Page 5 of pubid 813043 had products without prices (Bottega catalog, prepared-foods) AND `hotspots.length === 0 continue` short-circuited the page-sweep on the new pubid which would have rescued things | Removed the early `continue`, switched to correct pubid 824458 (discovered via city-specific page) |
| Vision sweep returned 0 products on subsequent dry-run | Pubid 813043 was a dead Bottega catalog (food-court items without prices), not the weekly flyer | Switched env var to current pubid 824458, added `ESSELUNGA_SHOPFULLY_CITY=verona` to enable city-fallback discovery |
| Cleanup chain-wide wiping fan-out from sibling pubid | `DELETE FROM offers WHERE campaign_id = X` (storeId NULL branch) ran on every multi-pub iteration | Added priority branch in `ingest.ts`: when `runtimeTargetStoreIds` is present, scope cleanup to `store_id = ANY(target)` |
| First run timed out at 20min during second pubid | Default `timeout 1200` killed node mid-vision; vision parallel for 208 hotspots @ concurrency=4 + ~118 transient errors retried each | Re-ran pubid 824460 alone with `timeout 1800`; completed in 775s |
| Trigger fired UPDATE on 40 sibling registry rows for one Famila SKU | Match was `(brand=NULL, name="Sottocosto Daya...", qty=NULL, ean=NULL)` — 40 sibling SKUs share that nameset | Accepted as expected behavior; the rows are idempotent UPDATE-to-same-URL, no row count growth |

## Final State

- **Fleet** (8 agents): all healthy, Daily memory ENOENT silenced, MC Telegram delivery working, MC Morning Todo new daily job at 07:10, morning cron jobs staggered 06:30 → 07:30.
- **SpesaBot working tree**: clean, 15 commits today (`d9bf3ea` … `edc781c`), production running on committed code.
- **Esselunga**: 327 store-bound offers across 3 Verona stores (was 24 stale chain-wide), discoverable via "Vicino a me" from Bussolengo.
- **Expiry feature**: `/api/deals/expiring` endpoint, badge on every product card, dedicated sidebar screen with chip selector.
- **Image registry**: 32 564 seed rows, automatic trigger-fed updates, 520 SKUs healed cross-chain on first backfill, lifecycle integrated into `run-pipeline.sh`.
- **Coverage uplift**: Esselunga images 73% → 85%, MD 87% → 95%, Lidl 70% → 100% (active offers), Conad 90% → 99% (active offers).

## Open Questions

- **Skill GWS path EACCES** (`/root/googleworkspace-cli/skills/...`) — known structurally, no fix applied today. Worth a follow-up to use real user paths (`/home/kos/...`) in the skill registry.
- **Despar / Aldi private-label coverage** — these chains have unique product ranges with no cross-chain match. The image registry can't help; would need a chain-specific image-extraction parser improvement.
- **`original_price` / `discount` parsing** for text-based chains (conad/despar/rossetto/lidl) — parsers rarely capture the strikethrough price. Out of scope for this session; per-chain regex work needed.
- **`store_id` binding for non-fan-out chains** — only migross/famila/conad-flyer/esselunga have per-store offers today. The multi-pub pattern from Esselunga generalizes but other chains may need different anchor data (e.g. Conad requires a separate store-discovery pipeline).
- **Esselunga first run had 118 vision errors of 200 hotspots** — Gemini 503 day correlated. Acceptable rate-limit retries handled it (87 net products), but if it recurs, sweep coverage suffers more than hotspot vision.
- **Poli with app strategy (mitmproxy + APK)** — deferred. Time-boxed 1-day investigation scheduled when user returns from visual UI verification.
