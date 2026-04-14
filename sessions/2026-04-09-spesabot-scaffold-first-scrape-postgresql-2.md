---
title: "SpesaBot — Project scaffold, PostgreSQL schema, first successful scrape + ingest"
date: "2026-04-09"
author: "kos-domus"
status: "processed"
tags: ["mcp", "mcp-servers", "automation", "configuration", "setup", "multi-agent", "testing"]
openclaw_version: "2026.4.8"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Scaffold SpesaBot as a real product, install PostgreSQL, design and apply the database schema, build the ETL ingest pipeline, run the first manual supermarket scrape end-to-end into the database.

## Context
- Deep-research pipeline fully operational (14 MCP tools, Playwright + Firecrawl, GLM-5.1 + Gemini + vision OCR)
- Co-CEO product plan approved: Verona province, 5-8 chains, Telegram bot, personalized savings engine
- Backend Expert and Orchestration Architect delivered detailed designs for schema, pipeline integration, and orchestration
- OpenClaw updated to 2026.4.8 (security advisory patches)

## Steps Taken

### 1. Updated OpenClaw 2026.4.5 → 2026.4.8
Security update patching: pairing/node scope escalation, env injection, trust-boundary injection, browser SSRF bypass, ClawHub integrity.

```bash
npm install -g openclaw@2026.4.8
```

Updated systemd service version and description. Gateway restarted successfully, all agents healthy.

**Result**: OpenClaw 2026.4.8 running, Telegram + WhatsApp channels online.

### 2. Consulted Backend Expert + Orchestration Architect
Both worked in parallel on detailed designs:

**Backend Expert delivered:**
- Full PostgreSQL DDL (13 tables + 2 views)
- Pipeline integration: JSON ETL via filesystem (deep-research writes JSON → SpesaBot ETL ingests to PostgreSQL)
- Deep-research optimizations: array extraction mode, viewport-scroll capture, Italian-specific prompts
- Complete Python code for price normalization (quantity parser, price parser, mechanic decomposition, reference pricing)

**Orchestration Architect delivered:**
- systemd timer (Wed 03:00 CET), NOT OpenClaw cron
- Sequential chain processing with 30s cooldown, retry on failure
- One job per chain for failure isolation
- RAM protection: MemoryMax=2G, closeBrowser() between chains
- Telegram alerts to Kos (critical/warning/info levels)
- Separate Telegram bot (grammY), NOT an OpenClaw agent

**Key agreement:** Both experts independently chose JSON ETL + direct pipeline import (not MCP calls) as the integration pattern.

### 3. Installed PostgreSQL 16 + PostGIS
```bash
sudo apt-get install postgresql postgresql-contrib postgresql-16-postgis-3
```

Created database, user, and extensions:
- Database: `spesabot`
- User: `spesabot` (password in local config only)
- Extensions: `postgis`, `pg_trgm`

**Result**: PostgreSQL 16.13 + PostGIS 3.4 running.

### 4. Created SpesaBot project structure
```
~/job-desk/spesabot/
├── db/
│   ├── 001_schema.sql        # Full schema DDL
│   └── 002_seed_chains.sql   # 8 Verona chains
├── src/
│   ├── db.ts                 # PostgreSQL connection pool
│   ├── normalize.ts          # Price/quantity/mechanic parsers (TypeScript port)
│   ├── ingest.ts             # JSON → PostgreSQL ETL
│   ├── runner.ts             # Pipeline orchestrator (per-chain, retry, alerts)
│   └── alerts.ts             # Telegram notification module
├── package.json
└── tsconfig.json
```

Dependencies: `pg`, `grammy`, TypeScript.

### 5. Applied database schema
13 tables created: chains, stores, products, product_skus, flyer_campaigns, offers, price_history, users, shopping_lists, shopping_list_items, staging_raw_offers + 2 views (active_offers_view, best_current_price_view).

Seeded 8 chains: Martinelli, Lidl, Eurospin, Famila, Conad, Pam, Despar, Aldi.

Fixed: `product_skus.product_id` changed to nullable (canonical product matching happens later, Phase 1 stores unmatched SKUs).

**Result**: Schema applied, 8 chains seeded.

### 6. Added PostgreSQL MCP server
Added `@modelcontextprotocol/server-postgres` to `.mcp.json`:
```json
"postgres": {
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://spesabot:...@localhost:5432/spesabot"]
}
```

**Result**: PostgreSQL queryable from Claude Code and agents.

### 7. Updated deep-research for array extraction mode
Added `extractionMode: 'array'` and `maxItemsPerPage: 50` to DomainConfig type and supermarket-deals.yaml. Updated `buildExtractionPrompt` in `llm-extract.ts` with array-specific rules and Italian supermarket conventions.

**Result**: LLM now returns arrays of products from multi-product pages.

### 8. Tested supermarket scraping — found working sources
Tested multiple sources:

| Source | Result | Issue |
|--------|--------|-------|
| doveconviene.it overview pages | 0 products | Shows flyer thumbnails, not product data |
| eurospin.it/offerte/ | 404 | URL changed |
| eurospin.it/volantino/ | 0 products | Embedded JS flyer viewer, no HTML product data |
| lidl.it/offerte | 404 | URL changed |
| lidl.it/offerte-settimanali | 4386 chars but category nav only | No product prices in HTML |
| **promoqui.it/volantini/eurospin** | **25 products** | **Real product data in HTML** |

**Key discovery**: PromoQui is the best source for Phase 1 — it exposes structured offer data in HTML, not just flyer images.

### 9. First successful end-to-end scrape + ingest
```
PromoQui (Eurospin) → Playwright (3.5s, free)
  → GLM-5.1 array extraction (25 products)
  → SpesaBot ETL ingest
  → PostgreSQL: 25 offers, 25 SKUs, 1 campaign
```

Products extracted with real data: Cola €0.79, Latte UHT €1.15, Formaggio Spalmabile €1.79, up to Necchi Tagliacuci €159.99.

**Result**: Full pipeline working from scrape to database.

## Configuration Changes
- OpenClaw: 2026.4.5 → 2026.4.8 (systemd service + gateway)
- `~/.config/systemd/user/openclaw-gateway.service` — version updated
- `~/job-desk/.mcp.json` — added PostgreSQL MCP server
- `~/job-desk/tools/deep-research/src/types.ts` — added `extractionMode`, `maxItemsPerPage`, `forceVision` to types
- `~/job-desk/tools/deep-research/src/normalizer/llm-extract.ts` — array extraction prompt, Italian supermarket rules
- `~/job-desk/tools/deep-research/configs/supermarket-deals.yaml` — added `extractionMode: array`, `maxItemsPerPage: 50`
- `~/job-desk/spesabot/` — new project: db schema, ETL, runner, alerts, normalize modules
- PostgreSQL 16 + PostGIS 3.4 installed and running

## Key Discoveries
- **PromoQui is the best source for Phase 1**: Unlike doveconviene.it (flyer thumbnails) and direct retailer sites (JS viewers), PromoQui exposes actual product names and prices in HTML text. This makes text-based LLM extraction work well.
- **Direct retailer sites are hard to scrape**: eurospin.it and lidl.it both use embedded flyer viewers or JS-rendered widgets. URLs change frequently (multiple 404s). Vision OCR is needed for these sites.
- **doveconviene.it overview pages show no product data**: The 3,500+ chars of HTML are all navigation links and metadata. Actual offers are in embedded flyer image viewers that require clicking through.
- **Array extraction mode works**: GLM-5.1 successfully returned 25 products as a JSON array from a single page, with proper Italian product names and prices.
- **PostgreSQL product_id must be nullable**: Phase 1 has no canonical product graph yet. SKUs are created without matching to canonical products — matching comes in Phase 1.5.
- **SAVEPOINT pattern needed for batch inserts**: When one product fails to insert inside a transaction, the entire transaction is aborted in PostgreSQL. Using SAVEPOINT/ROLLBACK TO SAVEPOINT allows skipping bad rows without losing good ones.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| eurospin.it/offerte/ → 404 | Retailer changed URL structure | Used eurospin.it/volantino/ (found via testing) |
| lidl.it/offerte → 404 | Same | Used lidl.it/offerte-settimanali |
| doveconviene.it: vision OCR returned empty array | Overview page shows thumbnails, not products | Switched to PromoQui as primary source |
| Ingest: "current transaction is aborted" | Failed INSERT rolled back entire transaction | Added SAVEPOINT/ROLLBACK TO SAVEPOINT per product |
| product_skus NOT NULL violation on product_id | Products table empty, similarity subquery returns NULL | Made product_id nullable, insert NULL for unmatched SKUs |

## Final State
- **OpenClaw**: 2026.4.8, all agents healthy
- **PostgreSQL**: 16.13 + PostGIS 3.4, spesabot database with 13 tables
- **SpesaBot project**: Scaffolded at ~/job-desk/spesabot/ with ETL, runner, alerts, normalize modules
- **First scrape**: 25 Eurospin products from PromoQui → ingested into PostgreSQL
- **Data**: 25 offers, 25 SKUs, 8 chains seeded, 1 campaign loaded
- **MCP**: PostgreSQL server added to .mcp.json (4 MCP servers total)
- **Deep-research**: Array extraction mode working for multi-product pages

## Open Questions
- Week 2: FastAPI + Telegram bot (SpesaBot user-facing interface)
- PromoQui as primary source: need to test Lidl, Conad, Famila, Pam pages on PromoQui
- Canonical product graph: when to seed 300-500 products and start matching SKUs
- PromoQui may also have anti-scraping — monitor for blocks
- systemd timer for weekly automation (defined in design, not yet created)
- Telegram bot token: need to create SpesaBot bot via BotFather
