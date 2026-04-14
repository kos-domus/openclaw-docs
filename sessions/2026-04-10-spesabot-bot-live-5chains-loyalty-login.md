---
title: "SpesaBot — Telegram bot live, 5 chains (154 offers), loyalty cards, persistent login setup"
date: "2026-04-10"
author: "kos-domus"
status: "processed"
tags: ["mcp", "mcp-servers", "automation", "configuration", "setup", "testing", "security"]
openclaw_version: "2026.4.8"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Build SpesaBot Week 2: Fastify API + Telegram bot, add loyalty card features, scrape 5+ chains into PostgreSQL, set up persistent browser login for authenticated chain sites.

## Context
- SpesaBot Week 1 completed: PostgreSQL schema, ETL ingest, pipeline runner, 93 offers from 2 chains
- Deep-research pipeline operational with array extraction mode
- Bot token created via BotFather
- Famila promo pages discovered as excellent direct data source

## Steps Taken

### 1. Evaluated Gemma as local LLM alternative
Checked if running Gemma locally via Ollama could replace OpenAI API fallback. Conclusion: not worth it.
- gemma-3-4b fits in RAM (~3GB) but quality is medium and speed is slow (30-60s)
- OpenAI API fallback costs ~€0.05/week when Z.AI is down (rare)
- Decision: keep current stack (GLM-5.1 → Gemini CLI → OpenAI), don't install Ollama

### 2. Built Fastify API (TypeScript)
API server at port 3080 with 6 endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /api/status` | System health + offer stats |
| `GET /api/search?q=` | Fuzzy product search across chains |
| `GET /api/deals/top` | Top discounts this week |
| `GET /api/chain/:slug` | Offers for a specific chain |
| `GET /api/compare?q=` | Price comparison across chains |
| `GET /api/chains` | List chains with active offer count |

All queries use PostgreSQL `pg_trgm` for fuzzy matching.

**Result**: API tested and working — search "yogurt" returns 17 results across chains.

### 3. Built Telegram Bot (grammY)
SpesaBot Telegram bot with commands:

| Command | Function |
|---------|----------|
| `/start` | Onboarding message |
| `/cerca [prodotto]` | Product search |
| `/offerte` | Top 10 deals of the week |
| `/catene` | Chains with inline keyboard buttons |
| `/confronta [prodotto]` | Price comparison |
| `/carte` | Loyalty card signup links |
| `/stato` | System status |
| Free text | Auto-search (type "pasta" → get results) |

Bot is deterministic (SQL queries, no LLM) — fast and zero cost per query.

**Result**: Bot live on Telegram, responding to commands with real data.

### 4. Added loyalty card features
Extended chains table with `loyalty_card_name`, `loyalty_signup_url`, `loyalty_app_url`.

Configured for: Famila (Carta Insieme), Lidl (Lidl Plus), Conad (Carta Insieme Conad), Despar, Pam (Carta Pam).

Bot shows 🔑 indicator for loyalty-only offers + `/carte` command with signup buttons (inline keyboard with URLs).

### 5. Scraped Famila — 3 stores, 68 offers
Famila promo pages (promo.famila.it) are the best data source found:
- Structured HTML with product names, original/offer prices, discount %, weights, price/kg
- 3 stores in Valpolicella area: Sant'Ambrogio, San Pietro in Cariano, Negrar
- 68 total offers with 37.1% average discount
- Top deals: Ravioli -50%, Müller yogurt -48%, Monini olio EVO -40%

### 6. Added Lidl — 15 offers
Discovered Lidl's weekly offer pages have dynamic URLs:
- Pattern: `lidl.it/c/super-offerte-kw-XX-YY/aZZZZZZ`
- Must discover from homepage navigation each week
- Excellent HTML data: prices, discounts, weights, price/kg
- 15 offers with 28.1% average discount

### 7. Added Conad — 32 offers
Used "Bassi e Fissi" (permanent low prices) page:
- URL: `conad.it/prodotti-e-marchi/bassi-e-fissi`
- 32 everyday products with fixed low prices
- Not weekly promotions — stable prices valid until month end
- Store-specific flyers require login (addressed in persistent profile setup)

### 8. Pam — 14 offers from PromoQui
PromoQui aggregator worked for Pam Panorama.

### 9. Aldi and Despar — not yet working
- **Aldi**: Fully JS-rendered (272 chars HTML), vision OCR screenshot captured but Z.AI 429 prevented extraction. Correct URL found: `/it/offerte-settimanali/offerte-di-questa-settimana.html`
- **Despar**: Cloudflare protection blocks scraping. PromoQui has too little data (1,462 chars).
- Both need authenticated access via persistent browser profile.

### 10. Created chain extraction procedures documentation
Created `configs/chain-sources.yaml` documenting for each chain:
- Working URLs, broken URLs, URL patterns
- Extraction method (text, vision, text+api)
- Quality rating, average products per page
- Gotchas (URL changes weekly, login required, JS rendering)
- Loyalty card info

### 11. Built persistent browser login system
Created `scripts/login-setup.ts` for authenticated scraping:
- Opens visible Chromium browser with persistent profile
- User logs in manually to each chain site
- Cookies/sessions saved to `~/.spesabot/browser-profile/`
- Pipeline reuses saved sessions for weekly automated scraping

Updated Playwright fetcher to detect and use persistent profile when available.

Service account: `kos.domus@gmail.com` (dedicated bot email, not personal).

**Status**: Code ready, waiting for Kos to register accounts on chain websites.

## Configuration Changes
- `~/job-desk/spesabot/src/api/server.ts` — new: Fastify API with 6 endpoints
- `~/job-desk/spesabot/src/bot/bot.ts` — new: grammY Telegram bot with 7 commands + free text search
- `~/job-desk/spesabot/scripts/login-setup.ts` — new: interactive login for persistent browser profile
- `~/job-desk/spesabot/package.json` — added: fastify dependency, api/bot scripts
- `~/job-desk/tools/deep-research/src/fetchers/playwright.ts` — added persistent browser profile support
- `~/job-desk/tools/deep-research/configs/chain-sources.yaml` — new: documented extraction procedures per chain
- `~/.openclaw/op-env-cached.sh` — added: `SPESABOT_BOT_TOKEN`
- PostgreSQL: added `loyalty_card_name`, `loyalty_signup_url`, `loyalty_app_url` columns to chains table

## Key Discoveries
- **Famila promo.famila.it is a goldmine**: Best structured data of any chain. HTML contains product names, both prices, discount %, weights, price/kg. No vision OCR needed. Different stores may have different promos.
- **Lidl URLs change weekly**: Pattern is `/c/super-offerte-kw-XX-YY/aZZZZZZ`. Must navigate homepage to discover current week's offer page URLs. Can't hardcode.
- **Conad "Bassi e Fissi" vs weekly flyers**: The Bassi e Fissi page is publicly accessible and has 32+ products. Weekly store-specific flyers require login.
- **Chain websites are hostile to scraping**: Aldi (100% JS), Despar (Cloudflare), Conad (login required), Lidl (dynamic URLs). Only Famila has a genuinely scraper-friendly promo portal.
- **PromoQui is hit-or-miss**: Works well for Eurospin and Pam, but LLM extraction returns 0 products for Lidl, Conad, Aldi pages on PromoQui — different page layouts confuse the extraction.
- **Gmail alias is simpler than a new account**: `kos.domus@gmail.com` for the bot's service accounts — all verification emails land in the existing inbox.
- **TypeScript end-to-end pays off**: Same language from scraping (deep-research) → ETL (ingest) → API (Fastify) → bot (grammY). Easy code sharing, shared types, natural path to React Native later.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| API port 3080 EADDRINUSE on restart | Old process still holding port | `fuser -k 3080/tcp` before starting |
| Bot module not found (wrong directory) | Background `&` loses `cd` context | Use `run_in_background` or absolute paths |
| Z.AI 500/429 throughout the day | Z.AI server instability | Fallback chain caught it (Gemini → OpenAI) |
| Lidl offer pages: 0 prices in HTML | Index page shows categories, not products | Discovered weekly-specific URLs from homepage links |
| Aldi 404 on old URL | URL pattern changed from `/offerte/` to `/it/offerte-settimanali/` | Navigated homepage to find current URL pattern |
| Despar Cloudflare block | Anti-bot protection | Need persistent browser profile with real session |
| Conad store flyers behind login | Login required for personalized offers | Persistent profile + service account registration |

## Final State
- **SpesaBot Telegram bot**: Live and responding
- **API**: Running at localhost:3080, 6 endpoints
- **Database**: 154 offers across 5 chains (Famila 68, Conad 32, Eurospin 25, Lidl 15, Pam 14)
- **Chain sources documented**: chain-sources.yaml with proven procedures per chain
- **Persistent login system**: Built, ready for account registration
- **Loyalty cards**: 5 chains with signup links, 🔑 indicator in bot

## Open Questions
- Register kos.domus@gmail.com on Conad, Despar, Aldi and run login-setup.ts
- Aldi extraction: even with correct URL and cookies, the page may need specific wait-for-content logic
- Despar Cloudflare: persistent profile may bypass it (real browser with cookies)
- Weekly automation: systemd timer not yet created — manual scraping for now
- Dedup across Famila stores: same product at same price appears 3x (one per store)
- Lidl URL discovery: need automated homepage scraping to find current week's offer URLs before the main scrape
