---
title: "SpesaBot — image library extension (Despar vision + fuzzy match + manual fill workflows) + MC Todo fixes (pipeline timeout, frontend micro-bugs, /admin/health)"
date: "2026-05-01"
author: "kos-domus"
status: "ready"
tags: ["automation", "cron", "scheduling", "debugging", "troubleshooting", "configuration", "remote", "tailscale", "cli", "skills", "git", "security"]
openclaw_version: "2026.4.24"
environment:
  os: "Linux 6.17.0-22-generic"
  ide: "VSCode native extension"
  model: "claude-opus-4-7[1m]"
---

## Objective

Two interleaved workstreams driven by the previous day's audit + this morning's MC Daily Briefing todo:

1. **Image library completion** — bridge the gap on chains the cross-chain registry can't reach (Despar 1.5%, Aldi 24%, private-label-heavy retailers). Build vision-based extraction for Despar, fuzzy cross-parser match in the registry, and two manual fill workflows (CSV + interactive CLI) so the operator can fill the remaining tail by hand.
2. **MC Todo fixes** — the new MC Morning Todo job (created in the prior session, fired for the first time today) flagged 10 actionable items. Address every code/config-side item, defer business decisions and large architecture refactors.

## Context

- MC Morning Todo cron job (job id `0b88a6d1-...`) fired for the first time at 07:10 today. The 10-item list was the input for the second half of the session.
- Friday pipeline timer (Tue/Fri 03:00) had run overnight and partially failed: 11/12 chains ingested OK, MD killed mid-vision by systemd `TimeoutStartSec=3600`. CRAI correctly refused (Shopfully stale fix from the prior session worked as designed).
- Image library state at start: Despar 2.3% / Aldi 24.5% on active offers, much lower than the rest. The audit had recommended manual fill paths for the long-tail private-label products.
- User session started on Mac, requested SSH-via-Tailscale workflow guidance. Mid-session the mini-PC also experienced an unexpected power-down — recovered with no data loss thanks to PostgreSQL fsync-on-commit.

## Steps Taken

### 1. Despar iPaper vision-based image extraction
The Despar text parser (`despar-flyer-fetch.ts`) extracts product names + prices from the iPaper flipbook overlays but never captures product images — Despar SKUs sat at 1.5% native image coverage. Built `src/parsers/despar-images.ts`:

- Reuses `discoverDesparFlyerUrl()` to find the current flyer URL
- Opens the flyer in Playwright, **passively sniffs** the signed CloudFront URLs of every `Pages/N/Zoom.jpg` as the lazy-loader fetches them (iPaper assets 403 without the right session, sniffing is the cleanest path in)
- Sends each page image to Gemini 2.5 Flash with a prompt asking for products + bounding boxes (normalized [0..1]) + brand/qty/price
- Crops each bbox via `sharp`, saves under `data/product-images/despar/<flyer-slug>/<hash>.jpg`, served by the existing `/product-images/` static mount
- Companion CLI `src/backfill-despar-images.ts` calls `register_product_image()` for each crop so the registry persists them indefinitely

**Result**: 131 products extracted from the 15 useful pages of the current Despar flyer in 522s. Total Gemini cost: ~$0.02. All 131 registered in `product_image_registry` with `source_chain='despar'`.

### 2. Fuzzy cross-parser match in `lookup_product_image`
Initial backfill applied only 13 of 131 extractions. Investigation: vision-OCR'd names (`"sensodyne dentifricio assortito 75ml"`) didn't equal the Despar text-parser names (`"Spremi e gusta assortiti Plasmon"`). Same product, different word order, different brand prefix — exact match fails.

Migration `db/020_image_registry_fuzzy_match.sql`:
- Extended `lookup_product_image()` from 3-layer to 4-layer cascade: EAN → exact (brand+name+qty) → name-only → **pg_trgm similarity > 0.30** (same brand) or > 0.45 (cross-brand). Quantity guard prevents `75ml` matching `100ml`.
- Added GiST trigram index on `name_norm` (`gist_trgm_ops`). Without it the similarity scan was a 12+ minute full table scan; with it, sub-second.

**Result**: Despar coverage jumped 2.3% → 22.6% (+27 healed). Aldi spillover: 24.5% → 44.7% (+20pp because Aldi shares brands like Vileda/Amuchina with Migross/Famila that the fuzzy now bridges). Esselunga 85.3% → 94.2%, Conad 98.8% → 99.7%, Lidl 70% → 100% (active offers).

### 3. CSV-based image import workflow
Two scripts to fill the long tail by hand:
- `src/export-missing-images.ts` — generates `/tmp/missing-images-<chain>-<date>.csv` with: sku_id, brand, name, qty, best_discount_pct, active_offers, **search_google_images** (preloaded query URL), image_url (empty). Filtered to skip parser junk: `length BETWEEN 6 AND 90` and a regex against banner keywords.
- `src/import-images-from-csv.ts` — reads filled CSV, validates each `image_url` (HEAD request, then GET-range fallback, then magic-byte sniff for CDNs that lie on Content-Type), applies `UPDATE product_skus SET image_url`. Idempotent.

Workflow: `node dist/export-missing-images.js <chain>` → spreadsheet → fill image_url col → `node dist/import-images-from-csv.js <path>`. Trigger fires on each UPDATE so the manual entries enter the registry automatically.

### 4. SSH tunnel setup (Mac → mini-PC via Tailscale)
User flagged that the workflow runs on mini-PC, edits happen on Mac. Walked through the `~/.ssh/config` setup step by step — file already existed with `Host 100.121.6.60`, just appended an alias `Host spesify-mini` pointing to the same Tailscale IP. Added `ServerAliveInterval 60` after a session timeout caused mid-loop disconnect.

### 5. Interactive CLI `manual-image-fill.ts`
After the user reported "Numbers fa schifo" (spreadsheet broke when pasting URLs that auto-rendered as images), pivoted to a one-product-at-a-time CLI to bypass spreadsheets entirely:

- Lists active SKUs missing an image, ordered by `discount_pct DESC`
- Prints each with brand/name/qty/best_discount + a Google Images search URL (Cmd+click in Terminal opens the browser)
- Prompts for the URL, validates via HEAD + magic bytes, applies UPDATE
- Special inputs: `<empty>` skip, `?` reprint, `q` quit (saved-so-far stays), `s` screenshot mode

Hit a real-world content-type bug: Azure Blob Storage serves valid JPGs as `application/octet-stream`. Replaced the `image/*` content-type check with magic-byte sniffing (FFD8FF for JPEG, 89504E47 for PNG, RIFF...WEBP for WebP, GIF87/89 for GIF). Now any CDN with broken MIME headers passes as long as the bytes are real.

### 6. Screenshot upload mode (`s` command in CLI)
For multi-product offers ("burger gusti assortiti") where no single Google image fits, added screenshot upload via scp:
1. User types `s` in the prompt
2. Script tells them to scp from a second Mac terminal: `scp <screenshot> spesify-mini:/tmp/spesify-shot.jpg`
3. User hits ENTER
4. Script reads `/tmp/spesify-shot.jpg`, sniffs magic bytes (no extension trust), copies to `data/product-images/<chain>/manual/<sku>-<sha12>.<ext>`, generates `https://app.spesify.xyz/product-images/<chain>/manual/...`, runs UPDATE
5. Cleans `/tmp/spesify-shot.jpg` so the next iteration can't reuse yesterday's upload by accident

**Result**: User completed 11/30 Despar fills via URL mode in the first sitting; the 7-8 multi-product offers that had been skipped now have a clean path forward.

### 7. Cookbook + wrappers (chain-agnostic)
After confirming both URL and screenshot paths are fully chain-parametric (the `chain` arg flows from CLI → `ingestShotForSku(chain, skuId)` → CDN dir → public URL → DB), produced:

- `docs/manual-image-fill-cookbook.md` — one section per chain (12 chains) with copy-paste blocks for the SSH command and the scp template
- `scripts/fill.sh` — one-liner wrapper sourcing op-env-cached.sh and dispatching to `dist/manual-image-fill.js`
- `package.json` — added `npm run fill` for sessions already on the mini-PC shell
- `~/.claude/commands/spesify-status.md` — slash command `/spesify-status [chain]` that runs the live-count query via SSH and reports a markdown table + suggested next chain to attack

### 8. MC Daily Briefing todo — fixes from this morning's run

**P0 — Friday pipeline timeout investigation**: journalctl showed the pipeline hit `TimeoutStartSec=3600` exactly during MD vision (12/12, last chain). Raised to 5400s (90min). Re-ran MD only: 327 products / 270 hotspots in 515s, well within new headroom. Also raised `spesabot-matching.service` `TimeoutStartSec` from 600s to 2400s — the prior audit had flagged matching jobs killed mid-flight at batch 12/42.

**P0 — Frontend micro-feedback bugs**:
- `toggleSpinner()` was setting `style.visibility = 'open'` — `'open'` isn't a valid CSS value, AND the existing CSS toggles via `.active` class anyway. Net result: search spinner permanently invisible. Fixed to `el.classList.toggle('active', !!on)`.
- `promptAddWatch()` was unguarded against double-click on the "+ Aggiungi avviso" button — second click stacked a second `#watch-form` with the same input IDs, breaking `getElementById()`. Added bail-out + refocus.

**P1 — `/api/admin/health` endpoint**: Owner-only (gated by `SPESABOT_OWNER_TG_ID`, same pattern as `/api/analytics/summary`). Returns:
- Per-chain: active_offers, pct_image, last_campaign_update, loaded_active, failed_recent_7d, empty_recent_7d, **stale flag**
- Registry: total rows, unique URLs, contributing chains, new in 7d/30d
- Recent failed/empty campaigns (last 14 days) — surfaces what the no-clean-on-zero ingest guard is correctly catching

**P2 — openclaw-docs upstream alignment**: Already done by Cos's `Daily docs elaboration` cron at 07:30 (visible in `changelog/CHANGELOG.md` — processed 3 sessions, updated 5 docs, captured 2026.4.29 release notes). The binary install upgrade itself was deferred per the self-assessment note about needing smoke tests for restrictive tool profiles.

## Configuration Changes

| File | Change |
|------|--------|
| `~/.config/systemd/user/spesabot-pipeline.service` | `TimeoutStartSec` 3600 → 5400 |
| `~/.config/systemd/user/spesabot-matching.service` | `TimeoutStartSec` 600 → 2400 |
| `~/.openclaw/op-env-cached.sh` | (no change today; SPESABOT_ADMIN_CHAT_ID was added in prior session) |
| `~/.ssh/config` (Mac) | Added `Host spesify-mini` alias + `ServerAliveInterval 60` |
| `~/.claude/commands/spesify-status.md` | New slash command |
| `db/020_image_registry_fuzzy_match.sql` | pg_trgm fuzzy cascade in `lookup_product_image` + GiST index |
| `package.json` | `"fill": "node dist/manual-image-fill.js"` |
| 4 new TS files | despar-images, backfill-despar-images, export-missing-images, import-images-from-csv, manual-image-fill |
| `scripts/fill.sh` | New one-liner wrapper |
| `docs/manual-image-fill-cookbook.md` | New per-chain copy-paste cookbook |

## Key Discoveries

- **iPaper passive URL sniffing**: iPaper's signed CloudFront URLs reject direct curl with 403 even with the right Referer. The path that works is to open the flyer in Playwright and capture `Pages/N/Zoom.jpg` URLs as the page lazy-loads them — the signature comes via the active session.
- **Vision-OCR vs text-parser name divergence**: same product can produce wildly different names depending on which path captured it. `"sensodyne dentifricio assortito"` (vision crop) vs `"Spremi e gusta assortiti Plasmon"` (text-overlay). Exact-match registry was useless across this divide; pg_trgm similarity bridged it cleanly with quantity guard preventing wrong-size matches.
- **Spreadsheet "smart paste"** auto-renders pasted image URLs as inline images instead of keeping them as text. Numbers (and Excel) both do this. The escape hatch is `Cmd+Option+Shift+V` ("paste and match style") or pre-formatting the column as Text. For users who can't get past it, an interactive CLI sidesteps the entire spreadsheet layer.
- **Azure Blob Storage** (and many CDNs) serve valid image bytes with `Content-Type: application/octet-stream`. Trusting Content-Type alone rejects valid uploads. Magic-byte sniffing (first 12 bytes) is the source of truth for image format detection.
- **systemd `TimeoutStartSec` is real ceiling, not soft**: hit at exactly 60min for `Type=oneshot`, kills the entire process tree. For long-running pipelines, budget headroom — Friday's 11/12 chains succeeded but MD got terminated mid-vision because cumulative time crossed the limit. New rule: pipeline stages should have systemd timeouts ≥ 1.5× the worst-case observed duration.
- **Tailscale + SSH + scp + multi-terminal** is a smooth-enough workflow for "edit on Mac, execute on mini-PC" patterns. The blocker was a 5-min idle SSH timeout during long manual loops; `ServerAliveInterval 60` solves it transparently.
- **Custom slash commands** in Claude Code are pure markdown files at `~/.claude/commands/<name>.md` with optional frontmatter (`description`, `argument-hint`). The body is the prompt the agent receives; `$ARGUMENTS` placeholder gets replaced by the post-`/name` text. Project-local commands also work at `<project>/.claude/commands/`.
- **Cron job that fires for the first time** is a real test event. The MC Morning Todo job fired its first ever run today, the output drove half this session, and validated the design end-to-end (Obsidian Daily read, specialist Drive folders queried, todo synthesized, posted to the Job-desk Telegram group). The expected failure mode (chat-not-found from yesterday's bug) didn't recur, confirming yesterday's `accountId: work` fix.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Despar image registry initial backfill applied only 13/131 entries | Vision-OCR'd names didn't equal text-parser names; exact (brand+name+qty) lookup failed | Added pg_trgm fuzzy fallback to `lookup_product_image` + GiST trigram index for performance |
| `lookup_product_image` 12-minute full table scan during backfill | Initial fuzzy run before trigram index was created | Killed the in-flight psql, created `idx_pir_name_trgm`, re-ran in <1min |
| `URL immagine ✗ rifiutato: content-type application/octet-stream` | Azure Blob Storage serves JPGs with generic MIME | Replaced strict content-type check with magic-byte sniff (always trusted for image format) |
| `ssh spesify-mini` from inside an SSH session failed: `Could not resolve hostname spesify-mini` | User was lanching from mini-PC shell, not Mac shell | Clarified prompt-vs-host conventions; documented that the SSH alias lives in Mac's `~/.ssh/config`, not the server's |
| `Read from remote host 100.121.6.60: Operation timed out. Connection to 100.121.6.60 closed.` mid-loop | Default SSH idle timeout closed the long-running interactive session | Added `ServerAliveInterval 60` + `ServerAliveCountMax 30` to the `Host spesify-mini` block (60s keepalive × 30 = 30min idle headroom) |
| Numbers auto-rendering pasted image URLs as inline images, breaking the CSV workflow | Spreadsheet "smart paste" defaults | Pivoted to interactive CLI (`manual-image-fill.ts`), bypassing spreadsheets entirely; documented `Cmd+Option+Shift+V` workaround as fallback |
| Mini-PC powered down unexpectedly mid-session | Unknown (pending journalctl `-k` check on next boot for thermal/OOM signals) | Verified post-recovery: 12 already-saved images preserved (PostgreSQL fsync), all systemd services restarted clean, no data loss |
| `spesabot-pipeline.service: start operation timed out. Terminating.` Fri 04:05 | `TimeoutStartSec=3600` exactly hit during MD vision (12/12 chain) | Raised to 5400s (90min). Re-ran MD manually: 327 products in 515s. |
| Search spinner permanently invisible | `style.visibility = 'open'` is not a valid CSS value, AND CSS expects `.active` class toggle | `el.classList.toggle('active', !!on)` |
| Watch-form duplicating on every "+" click | `insertAdjacentHTML('afterbegin', ...)` unguarded; second form stacked with shared input IDs | Added bail-out: if `#watch-form` exists, refocus `#watch-query` instead of inserting another |

## Final State

- **Image library**: Despar 22.6% (was 2.3%), Aldi 44.7% (was 24.5%), Esselunga 94.2%, Conad 99.7%, Lidl 100% on active offers. Registry has 32k+ rows including 131 vision-extracted Despar crops. Fuzzy match cascade in place + indexed.
- **Manual fill workflows**: 3 layers of shortcut available — copy-paste cookbook, `scripts/fill.sh <chain>` wrapper, `npm run fill <chain>` shortcut. Slash command `/spesify-status [chain]` for status checks.
- **Pipeline reliability**: timeouts headroom doubled. Friday's failed MD re-ingested. CRAI's stale Shopfully publication correctly rejected by yesterday's hard-fail guard (no junk data ingested).
- **Frontend bugs**: search spinner now visible; watch-form not duplicable.
- **Health visibility**: `/api/admin/health` owner-gated, returns per-chain freshness + registry growth + recent failures in one JSON.
- **MC Morning Todo cron**: fired its first run successfully at 07:10. Job validated end-to-end.
- **9 commits** on SpesaBot today (`f7fc2f4` through `a68fa57`, `76abb75`).

## Open Questions

- **Mini-PC unexpected power-down**: cause unknown. Need to check `journalctl -k | grep -i "thermal\|throttle\|out of memory"` on next reboot. Recurrence would justify investigating UPS / thermal mitigation.
- **OpenClaw binary upgrade 2026.4.24 → 2026.4.29**: deferred. Per the changelog self-assessment, requires smoke-testing every agent that uses a restrictive `tools.exec` / `tools.fs` profile because 2026.4.29 removes the implicit-widening behaviour. Schedule a maintenance window.
- **MC Todo P0 deferred items** still need user attention: Sacchitalia direction (SaaS vs module vs vertical), Sacchitalia business questions (volumes/deployment/BYOD/SLA), Mini App entrypoint redesign.
- **MC Todo P1 OrchArch refactors** deferred (out of scope for one sitting): runner.ts strategy modules, formal PG boundary contract for ingest/bot/API/ETL.
- **CRAI image coverage**: still 0 active offers because the only Shopfully publication is stale and was correctly rejected. Need to check whether CRAI has an alternative publication source or accept "CRAI dormant for this week".
- **Junk regex per chain**: the current banner-pattern regex in `manual-image-fill.ts`'s SQL is Despar-tuned (`tribù|app despar|happy card|...`). For other chains it's a no-op — won't break anything, but might leave some chain-specific banners ("Aldi Mobile", "Lidl Plus club") in the missing-image queue. Extend per-chain when noticed during real fills.
