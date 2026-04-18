---
title: "Spesify: nearby offers feed, store drill-down, watchlist notifications (Opus 4.7)"
date: "2026-04-18"
author: "kos-domus"
status: "ready"
tags: ["api", "automation", "web", "debugging", "testing", "scheduling", "configuration"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-7[1m]"
---

## Objective
Build three new features for Spesify to complete the discover → filter → notify loop: GPS-based nearby offers feed, store drill-down (tap store to see its offers), and product watchlist with Telegram notifications for registered users. Session started on Opus 4.6 for feature implementation, then switched to Opus 4.7 for verification.

## Context
Previous sessions built the Spesify Mini App with 9 sidebar sections, branded home screen, profile registration, search relevance, and chain info. Three features remained queued: nearby offer feed, store drill-down, and product reminders. PostGIS is already installed. All 130 stores have GPS coordinates. User profiles exist in the DB. Pipeline runs Tue+Fri 03:00, matching runs Tue+Thu+Sat 05:00.

## Steps Taken

### 1. Nearby offers API endpoint — `GET /api/offers/nearby`
Uses PostGIS `ST_DWithin` for efficient geo filtering.
- Accepts `lat`, `lng`, `radius_km` (max 50), `limit` (max 200)
- Returns offers from stores within radius PLUS all national offers (where `store_id IS NULL`)
- Each result includes a `km` field (distance to the store), null for national offers
- Ordered by `discount_pct DESC NULLS LAST`, then `offer_price ASC`
**Result**: Verona center 25km returns 8 store-specific offers from Sant'Ambrogio (13.6km), San Bonifacio (21.6km), Ronco (22.5km)

### 2. Store drill-down endpoint — `GET /api/store/:id/offers`
- Looks up store info first (returns 404 if not found)
- Returns offers where `sk.chain_id = store.chain_id AND (o.store_id IS NULL OR o.store_id = :id)`
- This correctly includes national offers from the same chain that apply to the store
**Result**: Famila San Bonifacio returns 555 offers, Migross stores return only national offers (correct, since Migross has no store-specific offers)

### 3. Watchlist DB schema — `db/007_user_watches.sql`
Created two tables:
- `user_watches`: FK to `user_profiles`, `watch_type` (product/category/keyword), `query`, optional `max_price`, `chain_filter[]`, `store_filter[]`, `is_active`, `last_notified_at`
- `watch_notifications`: tracks `(watch_id, offer_id)` pairs to prevent duplicate Telegram messages
- Indexes on `(user_profile_id, is_active)` and `(is_active, watch_type)`
- Grants on both tables + sequences to `spesabot` role
**Result**: Dedupe infrastructure in place, foreign keys cascade properly

### 4. Watchlist API — `GET/POST/DELETE /api/watches`
- `resolveProfileId()` helper: tries HMAC auth first, falls back to `telegram_user_id` from query/body
- GET lists active watches for the authenticated user
- POST validates `watch_type` enum and query length ≥ 2
- DELETE scopes by `user_profile_id` (can't delete others' watches)
- All endpoints return proper error JSON on bad input
**Result**: CRUD working, proper isolation between users

### 5. Notification daemon — `src/notify-watches.ts`
Standalone Node script that runs after each pipeline scrape:
- Loads all active watches joined with `user_profiles.telegram_user_id`
- For each watch, queries matching offers that haven't been notified yet (anti-join against `watch_notifications`)
- Match logic per type:
  - `product`: `ILIKE '%query%'` OR `similarity > 0.4`
  - `category`: `tag = ANY(sk.tags)`
  - `keyword`: `ILIKE '%query%'`
- Formats Markdown-safe Telegram message with product name, price, discount, chain, store
- Sends via Bot API `/sendMessage`, records notification to `watch_notifications` (ON CONFLICT DO NOTHING)
- 100ms delay between messages (rate limit: ~30 msg/sec)
- LIMIT 10 matches per watch per run to avoid spam
**Result**: Sends real Telegram alerts. Nutella watch sent 2 alerts, bio category sent 10 alerts during testing

### 6. Pipeline integration
Updated `scripts/run-pipeline.sh`:
- Changed `exec node runner.js` to regular invocation (so script continues after)
- Appended call to `notify-watches.js` with `|| echo "Notifications failed (non-fatal)"` for graceful failure
**Result**: Every Tue+Fri 03:00 pipeline run triggers notifications automatically

### 7. Mini App UI
Added two new sidebar items:
- "Vicino a me" (target icon) → GPS offer feed with "📍 Usa la mia posizione" button
- "I miei avvisi" (bell icon) → watchlist management

Store drill-down:
- Made store cards clickable (`onclick="showStoreOffers(id, name)"`)
- Added `#store-detail` container with back button
- Distance badge in coral on nearby results (`📍 X km`)
- Geolocation: tries Telegram `LocationManager` first, falls back to `navigator.geolocation`
- Proper error messaging (not "stuck loading")

Watchlist screen:
- Add-watch form: query + optional max_price
- List of active watches with "Rimuovi" button
- All entries render even if API fails (no auth required for localStorage-backed use)
**Result**: Clean 3-feature expansion, bumped cache version to `?v=37`

### 8. Opus 4.7 verification
After switching from Opus 4.6 to Opus 4.7, ran end-to-end verification on all three features:
- **Nearby**: 7 edge cases (invalid lat/lng, no coords, tiny radius, normal radius, wide radius)
- **Store drill-down**: 4 edge cases (invalid ID, non-existent ID, Famila with store offers, Migross with only national)
- **Watchlist**: 6 edge cases (no TG ID, invalid watch_type, too-short query, create, list, delete)
- **Daemon**: dedupe verified (nutella's 2 existing notifications did NOT re-send; bio added 10 new)
- **Auth**: attempted delete with wrong telegram_user_id → properly returns "Auth required"

Found + fixed one real bug: decimal radius (e.g. `radius_km=0.1`) failed with `invalid input syntax for type integer`. PostgreSQL was inferring integer type from `$3 * 1000`. Added `$3::float` cast.
**Result**: All 17 test cases passed after the float-cast fix

## Configuration Changes
- `db/007_user_watches.sql` — new tables: `user_watches`, `watch_notifications` + indexes + grants
- `src/api/server.ts` — added `/api/offers/nearby`, `/api/store/:id/offers`, `/api/watches` (GET/POST/DELETE), `resolveProfileId()` helper, `$3::float` cast in nearby query
- `src/notify-watches.ts` — new notification daemon
- `scripts/run-pipeline.sh` — removed `exec`, appended `notify-watches.js` call
- `dist/webapp/index.html` — new Nearby + Watches screens, sidebar items, cache version `?v=37`
- `dist/webapp/style.css` — made `.store-card` clickable (cursor, active transform)
- `dist/webapp/app.js` — `loadNearbyOffers()`, `showStoreOffers()`, `closeStoreOffers()`, `loadWatches()`, `addWatch()`, `deleteWatch()`, wired into `switchScreen()` title map

## Key Discoveries
- **PostgreSQL numeric parameters need explicit cast** — `$3 * 1000` where `$3` is a float value (e.g. `0.1`) triggers `invalid input syntax for type integer` because the `1000` literal is inferred as integer. Fix: `$3::float * 1000`.
- **PostGIS `ST_DWithin` on geography** is the right primitive for radius queries. Takes meters, so multiply km by 1000.
- **Telegram `LocationManager` API** works inside the WebView but needs explicit `init()` callback pattern. Falls back cleanly to `navigator.geolocation` for browser testing.
- **`watch_notifications` anti-join pattern** — `NOT EXISTS (SELECT 1 FROM watch_notifications WHERE watch_id=$1 AND offer_id=o.id)` is the clean way to implement "never re-notify". Paired with `INSERT ... ON CONFLICT DO NOTHING` on the unique `(watch_id, offer_id)` constraint, this is idempotent and atomic.
- **Non-exec pipeline wrapper** — removing `exec` from the final command in `run-pipeline.sh` allows chaining additional steps like notifications without needing a separate systemd unit.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| `invalid input syntax for type integer: "0.1"` on nearby query | PostgreSQL inferred `$3 * 1000` as integer context from the integer literal `1000` | Added explicit `$3::float * 1000` cast |
| Nearby 10km returned 0 store-specific offers | Famila's closest active store is 13.6km (Sant'Ambrogio); pipeline doesn't have offers for all 26 stores | Not a bug — correct behavior. Widened test to 25km to confirm data is there |
| `KeyError: 'count'` in Python test parser | Server returned 500 error object instead of result object (due to float bug above) | Fixed by the float cast |

## Final State
- **3 new API endpoints**: `/api/offers/nearby`, `/api/store/:id/offers`, `/api/watches` (CRUD)
- **2 new DB tables**: `user_watches` (watchlist), `watch_notifications` (dedupe)
- **1 new daemon**: `notify-watches.ts` auto-runs after every pipeline scrape
- **2 new Mini App screens**: Vicino a me, I miei avvisi
- **Store drill-down**: tap any store card to see its offers
- **All features verified end-to-end on Opus 4.7** — 17 test cases passing
- **1 bug found + fixed during verification**: decimal radius integer cast
- Services running: API, bot, pipeline timer, matching timer

## Open Questions
- **Notification volume throttling**: Currently sends up to 10 matches per watch per pipeline run. For users with broad watches (e.g. category=bio), this could be spammy. Consider daily digest or frequency cap.
- **Watch management UX**: No way to edit a watch (only create/delete). Fine for now; users just delete + re-add.
- **Category watches discoverability**: The add-watch form defaults to `keyword` type. Users might not know they can watch categories (bio, senza-lattosio, etc.). Add a dropdown.
- **Tosano**: `latuaspesa.com` still down. Check periodically.
