---
title: "Sabato monstre: OrchArch P1 chiuso (matching fix + monitor + etl_runs) + Spesify UI overhaul 4-slice (Lista+Avvisi unificati, card enriched, Esplora universale, drawer 12→5) + emoji fix + 2 spunti strategici (B2B + cloud) registrati"
date: "2026-05-03"
author: "kos-domus"
status: "ready"
session_type: "openclaw"
client: ""
openclaw_version: "2026.4.29"
environment:
  os: "Linux 6.17 (kos-domus mini-PC)"
  ide: "VSCode"
  model: "claude-opus-4-7[1m]"
tags: ["automation", "configuration", "debugging", "performance", "api", "memory"]
---

## Objective

Chiudere il **OrchArch P1** "health/fallback ingest/matching/fleet" su Spesabot (originato dal failure `spesabot-matching.service` 2026-05-02 con FK violation `product_id=5242`) splittato in 3 slice incrementali, e completare il **Spesify Mini App UI overhaul** sui 4 finding ancora aperti del Frontend Specialist 2026-04-27 ux-friction report (Reco #1 IA depth, #2 notifiche unificate, + 2 ulteriori richieste utente). Validare le fix su workload reale + registrare 2 spunti strategici futuri (B2B API + cloud migration) come memorie persistenti.

## Context

Stato di partenza: matching service failed da May 2 05:39 (FK violation in step 3 merge loop di `llm-match.ts`), pipeline trend sfavorevole (3 fail in 14 giorni), MC Daily Briefing del mattino flaggava 3 P0 ancora aperti (rerun matching, working tree sporco, micro-feedback Mini App). Spesify Mini App: 12 voci nel drawer + 4 home tile, navigation IA ancora troppo profonda per WebView Telegram, due sistemi notifiche paralleli che duplicano il modello mentale utente.

Frontend Specialist report del 27/04 elencava 6 finding (2 già fixed in commit 9995fbe Apr 30, 4 ancora aperti). User ha chiesto di portare a casa Reco #1 + Reco #2 + 2 enhancement extra (info store specifico nelle card, quick action add-to-list dalle card).

## Steps Taken

### 1. OrchArch Slice A — fix merge FK violation in `llm-match.ts`

Root cause analysis: il loop di merge in step 3 snapshotta `candidates.rows` PRIMA dell'iterazione. Quando merge #N elimina `product B`, una pair successiva referenziante B come `keep` side fallisce FK. Il singolo BEGIN/COMMIT rolla back tutti i 100+ merge validi precedenti.

Fix applicato:
- `Map<number,number>` redirect map: ogni merge registra `mergeId → keepId`
- `resolve()` walk-the-chain risolve ID attraverso merge precedenti
- Risoluzione di entrambi keepId/mergeId prima di ogni UPDATE
- Skip transitivo se `resolvedA === resolvedB` (`skipped_already_merged++`)
- `SAVEPOINT/RELEASE` per-merge: failure singola → ROLLBACK TO SAVEPOINT, increment `skipped_errors`, continua
- Summary line finale + asterisk marker `*` nel log se merge avvenuto via redirect (osservabilità)

Single-transaction model preservato (un BEGIN/COMMIT) → partial progress atomico; semantica shift da all-or-nothing a per-merge resilience.

**Result**: `tsc` build verde, dist regenerated, smoke su DB `0 orphan_skus` confermato. Commit `ca879f1`.

### 2. OrchArch Slice B — pipeline monitor + extend `/api/admin/health`

Catches il "service failed silently per 22h+" failure mode che aveva nascosto il fail di May 2.

Nuovo `src/monitor/pipeline-monitor.ts` (~150 righe):
- Polla 5 unit `spesabot-*.service` ogni 15min via `systemctl --user show`
- Diff su `ExecMainExitTimestamp` vs ultimo notificato per quel unit
- DM Telegram → owner con state, exit code, tail journal 12 righe
- **Idempotenza**: marker `lastNotifiedFailure` si setta SOLO dopo Telegram conferma 200 (se Telegram down → retry next tick)
- Auto-clear marker quando unit recovera
- State persistito in `~/.cache/spesabot-monitor/state.json`

systemd units nuove: `spesabot-monitor.service` (oneshot, env via op-env-cached.sh, 128M/20% CPU) + `spesabot-monitor.timer` (every 15min, OnBootSec=2min, Persistent).

API extension (`src/api/server.ts`):
- `/api/admin/health` ora ritorna `services[]` + `monitor_observed_at` + `monitor_stale` (true se >1h)
- API legge JSON file (no shell-out a systemctl) → se monitor giù → `monitor_stale=true` evidente

Verifiche: build OK, manual dry-run watcher con env vuote (5 unit detected, May 2 fail correttamente identificata, DM skippato per env vuote, marker NON impostato → retry funziona), state file pre-seeded col marker May 2 per non spammare DM al primo tick reale, timer enabled + first auto-run loggato `OK — 5 units checked, no new failures`. Commit `41fe863`.

### 3. OrchArch Slice C — `etl_runs` table + CLI + `/api/admin/runs` endpoint

Closes il visibility gap lasciato dal monitor (che riporta unit-state ma non per-script outcome metrics).

**Schema** (`db/021_etl_runs.sql`): `etl_runs(id, run_type, chain?, started_at, finished_at, status, summary jsonb, error_tail, pid)` + 2 indexes + view `v_etl_runs_recent` (adds `duration_sec` + `likely_stuck` >2h).

**Helper** (`src/etl-runs.ts`): library `startRun()/finishRun()` + CLI mode (`start --type X` / `finish --id N --status success --summary '{...}'`). Best-effort: errori del recorder non rompono mai la pipeline (returns `0` su start fail, swallow su finish).

**Wiring**:
- `canonical-match.ts`: tracking ai 2 exit path. Summary `{total_skus, matched, unmatched, products}`
- `llm-match.ts`: tracking ai 4 exit path (no candidates / no matches / success / failed). Summary `{total_pairs, llm_confirmed, merged, skipped_already_merged, skipped_errors, cross_chain_products}`
- `scripts/run-pipeline.sh`: trap-based start/finish via CLI, `SPESABOT_RUN_TYPE` env diff light/heavy. Summary `{chains, registry_before, registry_after}`
- `spesabot-pipeline-heavy.service`: aggiunto `Environment=SPESABOT_RUN_TYPE=pipeline-heavy`

**API**: `/api/admin/health` ora include `last_runs_by_type` (1 row per run_type, latest); nuovo `/api/admin/runs?run_type=&chain=&status=&limit=` (owner-only, max 200) per drilling history. CLI smoke-test passato (start → finish → row in `v_etl_runs_recent` con summary jsonb). Commit `cab725e`.

### 4. Validation matching service rerun (P0 dal MC briefing)

Reset failed state + lanciato `systemctl --user start spesabot-matching.service` in background (Type=oneshot, ~10 min wall sequenziale per 1428 candidate pairs / batch 30 = 48 LLM batch).

Risultati end-of-run:
- **matching-rule** (canonical): success in <1s, 37439 SKU matched, 11826 prodotti canonici, 0 unmatched
- **matching-llm** (Slice A fix validation): SUCCESS, summary `{total_pairs:1428, llm_confirmed:116, merged:113, skipped_already_merged:3, skipped_errors:0, cross_chain_products:203}`, duration_sec 2260

Il bug del 2 maggio è ufficialmente chiuso: 3 pair che avrebbero crashato con FK violation sono state skippate con grazia (transitively-merged detection del Slice A), 113 merge validi committati invece di rolled back, ZERO exception. Il monitor (Slice B) avrà clearato il marker `lastNotifiedFailure` al prossimo poll. etl_runs (Slice C) end-to-end: 2 righe nel DB con summary completi, conferma del flow tracking funzionante.

### 5. Spesify UI overhaul Slice 1 — Lista + Avvisi unificati (Frontend Reco #2)

Backend lasciato intatto (2 modelli distinti `user_watches` + `shopping_list_items`). Solo UI:
- Screen unificata "La mia spesa" con 2 tab: "Prodotti seguiti" + "Parole chiave"
- `screen-watches` ridotta a redirect shell (preserva muscle memory + deep-links)
- `data-screen="watches"` resolve via `switchScreen()` → `shopping-list` + `switchSpesaTab('watches')`
- Sidebar entry "I miei avvisi" rimossa (resta solo "La mia spesa")
- Home tile rinominata "La mia lista" → "La mia spesa"
- Nuovo CSS `.ms-tabs / .ms-pane` + `switchSpesaTab()` JS function

### 6. Spesify UI overhaul Slice 2 — Card offerta enriched (richieste utente)

Modifica `productCard()` in `app.js`:
- **Store-specifico**: invece di "12 negozi", mostra il negozio cheapest (name + city) + chip "+N altri" cliccabile per espandere lista completa con per-store offer prices (ordinati ASC). Backend già tornava `stores[]` in `/api/deals/top` e `/api/deals/expiring`.
- **Quick action bell**: nuovo button 🔔 stacked sopra il toggle "+/✓" lista. Click → inline `window.prompt` per max_price → `addWatch(productName, maxPrice)` (riusa endpoint `/api/watches` esistente, type=keyword)
- Nuovo CSS: `.product-actions` (stack vertical), `.product-alert-btn`, `.product-stores-more` (chip), `.product-stores-panel` (lista expandable)

### 7. Spesify UI overhaul Slice 3 — Search engine universale `/api/explore`

**Backend** (`src/api/server.ts:312+`): nuovo `/api/explore` endpoint che sostituisce `/api/search + /api/deals/top + /api/deals/expiring + /api/offers/nearby + /api/categoria/:tag` con un'unica API combinabile.

Filtri tutti opzionali: `q` (ILIKE su raw_name + canonical + brand), `chains` (csv), `store_ids`, `expiring_within_days`, `lat,lng,radius_km` (PostGIS ST_DWithin, national offers sempre incluse), `category` (tag canonical), `sort` (discount default | expiring | price | relevance auto se q). Response shape identica a `/api/deals/top` (results[] con stores[] aggregation) — productCard() renderizza unchanged.

Smoke test backend: q+chains+expiring combinati ✅, geo nearby ✅, category bio ✅, default top discount ✅. Endpoint esistenti restano live (no breaking change).

**Frontend**: nuovo `screen-explore` con search bar + 5 chip preset (Top sconti / In scadenza / Vicino a me / Categoria / Altri filtri) + active-filter strip rimovibile + sub-pickers per categorie e catene. State object (q, preset, chains[], expiringDays, geo, category) drive `/api/explore` con debounced refresh. Geo permission inline; nearby radius 20km default. CSS chip styles + active-filter pills.

### 8. Spesify UI overhaul Slice 4 — IA cleanup drawer 12 → 5

Drawer finale (5 voci):
- Home
- Esplora (assorbe Cerca + Offerte + Categorie + In scadenza + Vicino a me come filtri)
- La mia spesa (Lista + Avvisi tab)
- Negozi & carte (Negozi + Carte Fedeltà + Info Catene tab)
- Profilo (Account + Preferenze tab)
- Analytics (owner-only, hidden by default)

7 sidebar entries removed → empty redirect shell sections in HTML; `switchScreen()` redirects:
- `search/deals/expiring/nearby/categories` → `explore` con preset adatto
- `cards/chain-info` → `stores` + `switchNegoziCarteTab()`
- `settings` → `profile` + `switchProfileTab('settings')`

Nuovi tab switcher: `switchNegoziCarteTab()`, `switchProfileTab()`. Home tiles aggiornate per puntare a `explore` con `data-explore-preset` opzionale (vicino, expiring, category).

Single backend commit (`f1c4449`) + single frontend commit (`f56c9da`) per i 4 slice (sono UI atomic refactor, history più clean così).

### 9. Bugfix `\u{...}` literal escapes leaking come testo HTML

User report: chip preset Esplora mostravano `\u{26A1} Top sconti` letteralmente. JS string literal interpreta `\u{HEX}` come Unicode code point ma plain HTML lo treatta come literal. 11 occorrenze sostituite via Python `re.sub` con character reali (⚡ ⏱ 📍 🏷️ ⚙️ 📊 ➕). Cache-bust bumped. Commit `9724c8f`.

### 10. Spunti strategici registrati come memorie

User ha condiviso 2 spunti per future session (esplicitamente non da implementare ora):

1. **Spesify Business / B2B API**: monetizzare il DB offerte come servizio API per catene supermercati che vogliono visibilità competitor. Memoria `project_spesify_business_b2b.md` con razionale (marginal cost basso, ARPU 100-1000× consumer, mercato regionale meno saturo del globale) + 5 punti aperti pre-launch (legale/compliance scraping, trust paradox whitelisting, coverage gap geo, quality bar 99%+ con SLA, branding separato) + sequencing consigliato (POC pilot 1 catena → MVP API+dashboard → SDK TS/Python → enterprise).

2. **Migrazione cloud DB + VPS**: dal mini-PC home (SPoF) a stack managed (Neon/DO Postgres + Hetzner/Fly VPS + Cloudflare R2 storage). Memoria `project_migration_cloud_vps.md` con architettura target + gating financial (revenue B2B firmato prima di migrare per giustificare $30-80/mese baseline) + migration plan ~1-2 giornate (pg_dump → restore → DNS swap).

Entrambe cross-linked in `MEMORY.md`.

## Configuration Changes

- `~/job-desk/spesabot/src/llm-match.ts` — redirect map + per-merge SAVEPOINT (Slice A)
- `~/job-desk/spesabot/src/monitor/pipeline-monitor.ts` — nuovo (Slice B)
- `~/job-desk/spesabot/src/api/server.ts` — `/api/admin/health` esteso 2 volte (services + last_runs_by_type), nuovo `/api/admin/runs`, nuovo `/api/explore`
- `~/job-desk/spesabot/db/021_etl_runs.sql` — nuova migrazione (Slice C)
- `~/job-desk/spesabot/src/etl-runs.ts` — nuovo helper + CLI (Slice C)
- `~/job-desk/spesabot/src/canonical-match.ts` — wiring etl_runs
- `~/job-desk/spesabot/scripts/run-pipeline.sh` — trap start/finish via etl-runs CLI
- `~/.config/systemd/user/spesabot-monitor.service` + `.timer` — nuovi unit
- `~/.config/systemd/user/spesabot-pipeline-heavy.service` — added `SPESABOT_RUN_TYPE=pipeline-heavy`
- `~/job-desk/spesabot/src/webapp/{index.html, app.js, style.css}` — UI overhaul completo (4 slice + emoji fix)
- `~/.cache/spesabot-monitor/state.json` — pre-seeded marker May 2 fail per evitare DM spurio al primo tick

Spesabot commits (`~/job-desk/spesabot/`, repo local-only):
- `ca879f1` matching: redirect map + per-merge savepoint
- `41fe863` monitor: pipeline-monitor + extend /api/admin/health
- `cab725e` etl-runs: per-script run history + CLI + /api/admin/runs
- `f1c4449` api: /api/explore universal filter endpoint
- `f56c9da` spesify UI overhaul (Lista+Avvisi unificati, cards enriched, Esplora, drawer 12→5)
- `9724c8f` spesify: fix literal \u{...} escapes leaking as HTML text

Memorie aggiornate: `project_spesify_business_b2b.md` (nuovo), `project_migration_cloud_vps.md` (nuovo), `MEMORY.md` (2 pointer aggiunti).

## Key Discoveries

- **Single-transaction merge loop scala male su candidate snapshot stale**. Il pattern "snapshot + iterate + per-row UPDATE" senza redirect-map crea la categoria di bug "il record che voglio modificare adesso lo abbiamo già cancellato 50 iterazioni fa". Soluzione generale: o (a) re-query lo state dopo ogni mutation (RPC heavy), o (b) maintain in-memory redirect map (zero overhead). Approccio (b) qui è strict win: zero RT extra, semantica chiara, diagnostics asterisk `*` nei log.
- **Per-merge SAVEPOINT è il giusto trade-off tra resilience e atomicity**. Un transaction-level rollback su 1 errore distrugge 100+ merge validi (=> all-or-nothing). Per-row in transaction separate viola invarianti aggregate. SAVEPOINT dentro single BEGIN/COMMIT mantiene atomicità complessiva ma permette skip granulare delle pair patologiche.
- **Pattern "watcher monitor con pre-seeded marker" è critico per non spammare al deploy**. Quando installi un monitor di failure-state esistente, il primo tick deve "vedere" il fail come "già notificato" altrimenti spara un DM al deploy. Soluzione: dopo l'installazione, pre-seed manualmente lo state con il marker corrispondente al failure attuale. Da quel momento il monitor è in steady-state.
- **Universal search/explore endpoint con facet combinabili è 1-page mental model migliore di N-screen specializzate**. Il drawer di 12 voci nasceva da "1 screen per filtro". Riducendo a "1 screen Esplora con chip filter combinabili" abbiamo reso possibili query che prima erano impossibili (es. "Migross + scadenza 7gg + zona vicino"), eliminato 5 entry dalla nav, e simplified il mental model utente da "scegli quale screen" a "esplora poi modula con chip".
- **`window.prompt` funziona dentro Telegram Mini App WebView**, contrariamente a una vecchia gotcha documentata: oggi è used per il quick alert dialogue (richiesta max_price). Rotta inline sembra ok, ma se Telegram cambia restrizioni in futuro, fallback a inline form (come già fatto per `add-watch-btn`).
- **`\u{...}` literal in HTML è una bug-class facile da introdurre quando si copia-paste da un context JS**. JS string literal lo interpreta come Unicode code point, HTML lo treatta come testo letterale. Da memorizzare: in HTML usare il carattere reale o l'HTML entity (`&#x26A1;`), MAI lo sequence escape JS.
- **`.eml` drag&drop è un pattern di integrazione email leggero** che evita IMAP/Gmail API/OAuth complications. L'utente fa drag dell'email da Gmail web → file .eml → cartella locale → il watcher parsa MIME. Zero credenziali, multi-provider gratis.
- **`mailparser` ^3.9.8 ha solid coverage MIME** ma i types non espongono `partId` su `Attachment` (rimosso dal mio uso, fallback a random suffix per filename anonimo).
- **macFUSE su Apple Silicon è effettivamente proibitivo** per setup quick — Recovery Mode + Reduced Security forever è prezzo troppo alto per sshfs. SFTP via Cyberduck (drag&drop GUI) ha lo stesso effetto operativo senza compromessi security policy.
- **`mdpdf` (npm global) genera PDF da Markdown senza sudo** (no system tex install necessaria, wrappato puppeteer internamente). Utile per generare report sintetici senza pandoc/xelatex.

## Errors & Solutions

| Error | Cause | Solution |
|---|---|---|
| `error: insert or update on table "product_skus" violates foreign key constraint "product_skus_product_id_fkey"` (matching-llm step 3) | snapshot stale di candidate pairs + delete di prodotto referenziato in iterazione successiva | redirect map + per-merge SAVEPOINT, validato su matching reale (113 merge / 3 skip / 0 errors) |
| `Gemini response was not valid JSON (Xs). First 500 chars: ...` (extract pipeline) | default `maxOutputTokens` 8192 troncava JSON multi-record | alzato a 32768 in `generationConfig`, retry-once 503/429 in `callGemini()` |
| TS server IDE riportava "Cannot find name 'process'/'node:fs'" su edit di `extract.ts` | TS server stale del workspace | ignorato — `npm run build` reale (tsc) passava pulito, falsi positivi IDE |
| Chip preset Esplora mostravano `\u{26A1} Top sconti` come testo letterale | JS string literal escape inserito in HTML — interpretato come testo, non Unicode | replace via `re.sub(r'(?:\\u\{[0-9A-Fa-f]+\})+', sub, content)` con character reali (⚡ ⏱ 📍 🏷️ ⚙️ 📊 ➕) |
| macFUSE "system extension required ... could not be loaded" su Apple Silicon | macOS kernel-extension policy richiede Recovery Mode + Reduced Security | pivot a Cyberduck SFTP — stesso UX drag&drop GUI, zero kernel extension |
| Default polling watcher con sleep 60 → blocked da harness | runtime block su sleep > X for non-monitor pattern | usato `until grep -q watcher_exit /tmp/log; do sleep 5; done` invece di chained sleeps |

## Final State

- ✅ OrchArch P1 chiuso 100% (Slice A+B+C tutti committati e validati su workload reale)
- ✅ matching service: status `inactive (dead) status=0/SUCCESS` dopo rerun con 113 merge validi committati, monitor watchdog ha clearato marker
- ✅ Spesify Mini App: drawer da 12 a 5 voci, home da 4 a 5 tile JTBD, screen-explore con 5 chip preset combinabili, card offerte con store-specific + quick add-to-list/avviso, La mia spesa unificata, Negozi&carte + Profilo con tab interni
- ✅ Cache-bust webapp bumped 4 volte; tutti gli endpoint `/api/explore` + `/api/admin/runs` + `/api/admin/health` esteso live e auth-gated
- ✅ 6 commit Spesabot (ca879f1, 41fe863, cab725e, f1c4449, f56c9da, 9724c8f) — repo local-only no remote
- ✅ 2 spunti strategici registrati come memorie persistenti (B2B API + cloud migration), entrambi cross-linked in MEMORY.md

## Open Questions

- **GPT-5.5 stability monitoring** (continua dal session log Sat 2026-05-02): primi giorni con i 2 agent fleet su `gpt-5.5` primary. Da osservare p99 latency model-resolution + error rate vs `gpt-5.4` fallback. Soglia rollback manuale se regression >20% lat o errori in serie.
- **Backend/CSO/Inbox weekly first run** lunedì 4 maggio 04:00-09:00 — primi run reali dei job creati 27-28 aprile. Da verificare che girino senza errori di context/tooling.
- **MOC auto-promotion**: Evening Digest 2026-05-02 era il primo a tentare auto-promotion topic basata su 14gg rolling. Verificare nei log se ha proposto candidati corretti (e Gemini topic promotion che è già stata accettata da Rakki — vedi memoria `feedback_obsidian_topic_links.md` aggiornata a 13 topic).
- **Per-chain rows in `etl_runs`** (deferred): lo schema supporta `chain` nullable ma il wiring runner.ts è coupled all'OrchArch P1/L "extract strategy modules from runner.ts" refactor. Da fare insieme.
- **Spesify visual QA mobile** dopo l'overhaul: Rakki ha confermato emoji fix ma il visual end-to-end completo dei 5 nuovi screen merita un giro reale di interazione su mobile (annotations rosse if any → fix iterativo).
- **Spunto B2B**: prima decisione operativa è il 1° contatto con catena pilot (es. Despar Verona) — da cui derivano willingness-to-pay + features priorità. Da rimettere in agenda quando Rakki vuole tornare sul tema.
- **Spunto cloud migration**: gating revenue B2B firmato. Senza, infra resta on mini-PC.
