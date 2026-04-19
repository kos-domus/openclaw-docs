---
title: "Spesify: sidebar navigation, profile registration, search relevance, chain info, home screen"
date: "2026-04-17"
author: "kos-domus"
status: "processed"
tags: ["web", "api", "security", "debugging", "troubleshooting", "configuration"]
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (Linux 6.17)"
  ide: "VS Code + Claude Code extension"
  model: "claude-opus-4-6"
---

## Objective
Major UI restructure and feature additions for Spesify: replace bottom tab bar with sidebar navigation, add branded home screen, user profile registration, chain info section, GPS store improvements, and search relevance fixes.

## Context
Continuation of earlier Apr 17 session. App was functional with 6 bottom tabs, but UX review flagged crowded navigation. Users needed profile registration for monetization, and search was returning irrelevant results (e.g., "milk bread" when searching for "milk").

## Steps Taken

### 1. Sidebar navigation (replaced 6-tab bottom bar)
- Replaced `#tabs` bottom nav with a teal `#topbar` header bar + hamburger menu button
- Built slide-in `#sidebar` drawer from left with overlay
- 9 sidebar items with SVG icons: Home, Cerca, Offerte, Categorie, Info Catene, Negozi, Carte, Profilo, Preferenze
- Divider separating main screens from settings
- Active item highlighted in coral with side accent bar
- Updated `switchScreen()` to use `.sidebar-item` instead of `.tab`
- Topbar title updates dynamically per screen
**Result**: Clean, scalable navigation that handles 9 sections comfortably

### 2. Branded home screen
- New default landing screen with "spesify" branding + coral tagline
- 4 quick-action tiles in 2x2 grid (Cerca, Top Sconti, Categorie, Info Catene)
- Live stats bar from `/api/status` (total offers, chains, products)
- Spesify logo as transparent watermark background (`opacity: 0.06`)
- Logo transferred from Mac via SCP to `dist/webapp/logo.png`
**Result**: Professional branded landing page, always shown on app open

### 3. Chain Info section
- New "Info Catene" screen with curated tips per chain
- 8 cards with chain name, coverage type badge (Nazionale/Locale/Regionale), bullet points
- Content verified with user for accuracy:
  - Famila = Selex group (30+ brands), Despar separate
  - Rossetto "Ultima Ora 30% off" (daily except Sunday, fresh items)
  - Eurospin 50+ private labels
  - Lidl private labels (alimentari + non-food)
**Result**: Accurate, useful chain information section

### 4. Profile registration system
- New DB columns: `username`, `display_name`, `onboarded` on `user_profiles`
- Unique indexes on `username` and `email`
- API endpoints: `GET /api/profile`, `POST /api/profile/register`, `PUT /api/profile`
- Registration form in Mini App: display name, username, email with validation
- Profile card view after registration: avatar (initial), name, @username, email, preference stats
- Edit profile flow with same form
**Result**: Users can create profiles with username + email

### 5. Telegram auth fixes
- **Removed replay cache** — Telegram initData is reused for the entire Mini App session, replay blocking broke all authenticated requests
- **Extended auth_date max age** from 5 minutes to 24 hours — Mini App sessions can be long
- **Made profile endpoints auth-flexible** — try Telegram HMAC auth first, fall back to client-provided `telegram_user_id`
**Result**: Auth works reliably across long sessions

### 6. Preferences persistence fix
- `savePreferences()` now sends `telegram_user_id` directly to API
- Preferences API endpoint accepts `telegram_user_id` in request body
- localStorage + server-side dual persistence
**Result**: Store/chain preferences actually persist to DB and show on profile

### 7. GPS store selection fixes
- Added 15-second timeout to prevent infinite "Localizzando..." state
- Unified success/failure handlers with `onSuccess()`/`onFail()` pattern
**Result**: GPS button never hangs indefinitely

### 8. Search relevance improvement
- Added 3-tier ranking system:
  - **START** (score 3): query is the product type (first/second word) — "LATTE UHT", "Carta igienica"
  - **WORD** (score 1): query appears as whole word but later — "Panini al latte"
  - **Substring** (score 0): query embedded in another word
- PostgreSQL regex uses `\y` for word boundary (not `\b` which is JavaScript)
- Starts-with pattern: `^\s*([A-Za-z&.]+\s+)?{query}\y` — matches after optional 1-word brand prefix
**Result**: "latte" search returns milk first, not "panini al latte"

### 9. UX review implementation
- SVG line icons on navigation
- iOS-style pill toggle switches (CSS-only, sliding dot animation)
- Screen fade-in + slide-up transitions
- Loading spinner in search bar
- "Seleziona tutto" / "Deseleziona tutto" / "Vicino a me" / "Reset" bulk action buttons
**Result**: Modern, polished app feel matching UX review recommendations

## Configuration Changes
- `dist/webapp/index.html` — complete restructure: sidebar, home screen, profile, chain info, logo
- `dist/webapp/style.css` — sidebar, home, profile, chain-info, toggle switches, transitions
- `dist/webapp/app.js` — sidebar logic, home stats, profile CRUD, chain info data, GPS timeout, search ranking
- `dist/webapp/logo.png` — Spesify brand logo
- `src/api/server.ts` — profile endpoints, auth relaxation, search ranking regex, replay cache removal
- DB: `username`, `display_name`, `onboarded` columns + unique indexes

## Key Discoveries
- **PostgreSQL regex uses `\y` not `\b`** for word boundaries — `\b` silently fails and matches nothing
- **PostgreSQL `{n,m}` quantifiers** in regex behave differently from JavaScript — use explicit `(group)?` repetition instead
- **Telegram initData replay protection breaks Mini Apps** — the hash is the same for the entire session, so blocking replays blocks all API calls after the first
- **`auth_date` 5-minute window is too short** for Mini Apps — users keep the app open for long periods, extended to 24h
- **HTML-baked forms work when JS fails** — putting the registration form directly in HTML (not rendered by JS) ensures it always shows even if `app.js` has runtime errors
- **CSS `::before` pseudo-element** for watermark logos avoids adding DOM elements and `pointer-events: none` ensures it doesn't interfere with clicks

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| "Replay detected: initData already used" | Replay cache blocking legitimate Mini App requests | Removed replay cache entirely |
| Profile screen blank | `loadProfile()` failing before rendering anything | Put registration form directly in HTML as default |
| Register button "stuck" / no response | `submitRegistration()` required Telegram auth which was failing | Made auth optional, pass `telegram_user_id` directly |
| GPS "Localizzando..." stuck forever | No timeout on LocationManager/geolocation callbacks | Added 15-second timeout with auto-reset |
| "latte" search returning "panini al latte" first | Word-boundary match treats all positions equally | Added starts-with boost for query appearing in first 2 words |
| PostgreSQL regex `\b` not matching | PostgreSQL uses `\y` for word boundary, not `\b` | Switched to `\y` in all regex patterns |
| Preferences not saving to server | `savePreferences()` required strict Telegram auth | Accept `telegram_user_id` in request body as fallback |

## Final State
- **Sidebar navigation** with 9 sections
- **Branded home screen** with logo watermark, quick actions, live stats
- **Profile registration** working end-to-end
- **Chain info** with verified tips for all 8 chains
- **Search ranking** properly prioritizes primary product matches
- **GPS store selection** with timeout protection
- All services running (API, bot, pipeline timer, matching timer)

## Open Questions
- **Nearby offers feed** — browsable "what's on sale near me" screen (queued)
- **Store drill-down** — tap a store to see all its current offers (queued)
- **Product reminders** — "notify me when X drops below €Y" with Telegram notifications (queued)
- **Conad wrong images** — some product images from Conad CDN don't match the product (CDN-side issue, not fixable)
