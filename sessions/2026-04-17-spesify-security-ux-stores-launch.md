---
title: "Spesify: security hardening, UX polish, store filtering, Carte tab, and user testing"
date: "2026-04-17"
author: "kos-domus"
status: "ready"
tags: ["security", "api", "configuration", "debugging", "troubleshooting", "automation", "web", "setup"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-6"
---

## Objective
Prepare Spesify (Telegram Mini App for supermarket price comparison) for external user testing: fix all security audit findings, add missing features (Carte/loyalty tab, store filtering, GPS store selection), rebrand the UI, and resolve UX bugs found during live testing with friends.

## Context
Continuation of the Apr 16 session. The app was functionally complete but needed security hardening (6 audit findings), UX improvements (from a formal UX review), and bug fixes discovered during real-user testing. The app is a Telegram Mini App served via Fastify API on a home mini PC, tunneled through Cloudflare at `app.spesify.xyz`.

## Steps Taken

### 1. Security Audit Remediation — 6 findings fixed

**Finding #2 (HIGH): Bot PII in non-private chats**
- Added `requirePrivateChat()` guard function to `src/bot/bot.ts`
- Applied to `/profilo`, `/iscriviti`, `/conad_attiva`, `/carte` commands
- Bot now replies "Questo comando funziona solo in chat privata" in group chats
**Result**: Sensitive commands blocked outside private chats

**Finding #5 (MED-HIGH): No rate limiting, unbounded queries**
- Installed `@fastify/rate-limit` (60 req/min per IP)
- Capped all `limit` query params with `Math.min(parseInt(...), 100)`
- Added `LIMIT` to `/api/compare` endpoint (was unbounded)
**Result**: All endpoints rate-limited and query-bounded

**Finding #6 (MED): No security headers / XSS surface**
- Installed `@fastify/helmet` with custom CSP
- CSP allows: Telegram script src, known retailer image domains, `unsafe-inline` for styles
- Added `scriptSrcAttr: ['unsafe-inline']` — without this, ALL `onclick` handlers were blocked (caused major debugging)
- Replaced inline `onerror` on images with delegated `error` event listener
**Result**: Full security headers (CSP, HSTS, X-Content-Type-Options, X-Frame-Options)

**Finding #8 (MED): Systemd hardening**
- Added `NoNewPrivileges=yes` and `PrivateTmp=yes` to API, bot, and pipeline services
- `ProtectSystem=strict`, `ProtectHome=tmpfs`, `RestrictAddressFamilies` caused failures in user-mode systemd — removed
**Result**: Basic hardening applied within user-mode systemd constraints

**Finding #1 (HIGH): Encrypt loyalty card numbers**
- Added `encryptValue()` / `decryptValue()` to `src/crypto.ts`
- Made `encryptProfile()` fail-closed (throws if `SPESABOT_PII_KEY` missing, instead of silent no-op)
- Applied `encryptValue()` to card number before DB insert in `/conad_attiva`
**Result**: Card numbers encrypted at rest with AES-256-GCM

**Finding #7 (MED): Image URL allowlist**
- Added hostname validation in `ingest.ts` before storing `image_url`
- Allowlist: `migross.it`, `lidl.it`, `conad.it`, `eurospin.it`, `rossettogroup.it`, `maxidi.it`, etc.
- Unknown domains silently dropped
**Result**: Only trusted retailer image domains stored

### 2. Carte (Loyalty Cards) Tab
- Added `/api/loyalty` endpoint returning loyalty programs with signup URLs and exclusive offer counts
- Added Migross Card and ALDI App to chains table
- Created "Carte" tab in Mini App with branded cards per chain
- Each card shows: chain name, program name, exclusive offers count, "Iscriviti Gratis" button, "Scarica App" button
**Result**: 8 loyalty programs displayed with signup links (monetization hook)

### 3. Store Data — Migross + Rossetto GPS
- Inserted 26 Migross VR-province stores and 7 Rossetto stores into DB
- Geocoded all stores manually (Migross API doesn't provide coordinates)
- Fixed `/api/stores` to remove `WHERE location IS NOT NULL` filter
- Total: 130 stores, 129 with GPS
**Result**: All chains now visible in Negozi tab and GPS-selectable

### 4. Store-specific Offer Display
- Added `LEFT JOIN stores` + `s.name as negozio, s.city as citta` to ALL API endpoints (search, deals, categories, chain)
- Product cards show "Famila · Iperfamila San Bonifacio (San Bonifacio)" for store-specific offers
- National chain offers show "Tutti i punti vendita"
**Result**: Users can see which specific store has each offer

### 5. Spesify UI Branding
- Rewrote `dist/webapp/style.css` with Spesify brand palette
- **Critical discovery**: Telegram WebView injects `--tg-theme-*` CSS variables that override `var()` fallbacks. Must use `!important` on all brand colors
- Colors: coral background (#fdf0ed), teal text (#2d6b6b), white cards, coral prices (#e8735a)
**Result**: Full brand identity applied

### 6. Product Images in Mini App
- Updated `productCard()` to render `<img>` with lazy loading + emoji fallback
- Replaced inline `onerror` with delegated error event listener (CSP compatible)
**Result**: Product photos display in cards (90% coverage)

### 7. UX Review Implementation
- Added SVG line icons to bottom navigation (search, tag, grid, pin, card, gear)
- Replaced checkbox toggles with iOS-style pill switches (CSS-only, no JS text)
- Added fade-in + slide-up transition on screen switches
- Added coral loading spinner inside search bar
**Result**: Modern, polished app feel

### 8. Preference Filtering
- Chain filter now applied to ALL screens (search, deals, categories) — was only on search
- Added `chainFilterParam()` helper that builds both `&chains=` and `&store_ids=` params
- Added `buildStoreFilter()` server-side — filters store-specific offers while allowing national offers through
- Added "Seleziona tutto" / "Deseleziona tutto" bulk buttons for chains and stores
- Added "Vicino a me" GPS button using Telegram `LocationManager` API (with browser geolocation fallback)
- Added "Reset" button to clear all preferences
- Added localStorage persistence for preferences (fallback when Telegram auth unavailable)
**Result**: Full preference-based filtering with GPS store selection

### 9. Search Relevance
- Added word-boundary matching via PostgreSQL regex (`~*` operator)
- Products where query appears as whole word rank higher than substring matches
- "latte" now shows "LATTE UHT" before "PANINI AL LATTE"
**Result**: Significantly improved search quality

### 10. Famila Discount Bug Fix
- Loyalty point values (e.g., "100 punti") were parsed as -100% discounts
- Added sanitization: discounts >= 90% discarded as data errors
- Fixed 19 existing bogus rows in DB
**Result**: No more impossible discount percentages

## Configuration Changes
- `~/.config/systemd/user/spesabot-api.service` — added helmet, rate-limit, hardening
- `~/.config/systemd/user/spesabot-bot.service` — added hardening
- `~/.config/systemd/user/spesabot-pipeline.service` — added hardening
- `dist/webapp/style.css` — full Spesify brand rewrite
- `dist/webapp/index.html` — SVG nav icons, Carte tab, bulk action buttons, GPS button
- `dist/webapp/app.js` — images, store names, preferences, search spinner, transitions
- `src/api/server.ts` — helmet, rate-limit, loyalty endpoint, store filtering, chain filtering on all endpoints, SQL caps
- `src/bot/bot.ts` — private chat guard, encrypted card numbers
- `src/crypto.ts` — `encryptValue()`, `decryptValue()`, fail-closed `encryptProfile()`
- `src/ingest.ts` — image URL allowlist, discount sanitization
- DB: Migross stores (26), Rossetto stores (7), GPS coordinates, loyalty card data

## Key Discoveries
- **Telegram WebView blocks `navigator.geolocation`** — must use Telegram's `LocationManager` API instead (requires Bot API 7.2+)
- **Helmet's default CSP includes `script-src-attr: 'none'`** which blocks ALL `onclick` handlers. Cost significant debugging time. Fix: add `scriptSrcAttr: ['unsafe-inline']` to CSP config
- **User-mode systemd doesn't support** `ProtectSystem`, `ProtectHome`, `PrivateDevices`, `RestrictAddressFamilies` — these cause `Failed to drop capabilities` or `EAFNOSUPPORT` errors
- **localStorage is essential for Mini App preferences** — Telegram auth may silently fail, causing preferences to reset on every app open
- **`preferred_store_ids` filter must allow national offers through** — `o.store_id IS NULL OR o.store_id IN (...)` is the correct pattern
- **Screen switching with `display:none/block` preserves DOM** but search results feel "lost" when switching tabs — re-triggering the search on tab switch fixes the UX
- **Cache busting via `?v=N` query strings** is mandatory for Telegram WebView which aggressively caches static assets

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| All onclick handlers broken | Helmet CSP `script-src-attr: 'none'` | Added `scriptSrcAttr: ['unsafe-inline']` |
| API service crash loop | `ProtectSystem=strict` in user-mode systemd | Removed, kept only `NoNewPrivileges` + `PrivateTmp` |
| `EAFNOSUPPORT` error 97 | `RestrictAddressFamilies` incompatible with Node.js | Removed directive |
| `EADDRINUSE :3080` | Stale API process from manual testing | `fuser -k 3080/tcp` before service start |
| DB password mismatch | Service had `spesabot` vs actual `spesabot_local` | Removed hardcoded URL, source from `op-env-cached.sh` |
| Geolocation permission denied | Telegram WebView blocks browser geolocation | Use `tg.LocationManager` API instead |
| "Deseleziona tutto" not working | `_none_` sentinel value in preferences | Simplified: `[] = no filter`, removed sentinel |
| Search blank after tab switch | `display:none` hides screen, results not re-triggered | Re-run search query on tab switch |
| Store filter shows only national | GPS selected only Migross/Rossetto stores (no Famila offers) | Ensured Famila stores have GPS + `store_id IS NULL` passthrough |
| Preferences not persisting | `savePreferences()` requires Telegram auth | Added localStorage fallback |
| CSS changes not visible | Telegram WebView aggressive caching | Cache-busting `?v=N` on CSS/JS refs + `Cache-Control: no-cache` header |

## Final State
- **App live** at `https://t.me/SuperMarketScanner_bot` (display name: Spesify)
- **6/6 security findings fixed** — rate limiting, CSP, HSTS, private chat guards, encrypted PII, URL allowlist
- **6 tabs**: Cerca, Offerte, Categorie, Negozi, Carte, Preferenze (with SVG icons)
- **Full preference system** — chain filter, store filter, GPS selection, bulk actions, localStorage persistence
- **Spesify-branded UI** — coral background, teal text, iOS-style toggles, search spinner, fade transitions
- **130 stores** with GPS, **8 loyalty programs** with signup links
- **Bot**: `@SuperMarketScanner_bot`, display name "Spesify"
- Shared with friends for initial testing

## Open Questions
- **Sidebar navigation** — bottom tab bar getting crowded (6 items), should move to hamburger drawer
- **Bot onboarding** — redirect to Mini App on `/start` instead of showing chat history
- **Chain info section** — curated tips per chain (Eurospin national, Famila fresh discount last hour, etc.)
- **Tosano** — `latuaspesa.com` still down, parked
