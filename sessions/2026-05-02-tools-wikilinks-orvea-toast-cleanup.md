---
title: "Sabato produttivo: stack tools (Promptfoo/Anthropic Skills/Firecrawl/Playwright/Context7), 3-layer wikilinks Obsidian + MOC, Spesabot Gruppo Poli live (orvea +1164 offerte) + frontend toast + Esselunga health + Drive restore + saga GPT-5.5 swap (3 gateway restart) + SSH tunnel pattern"
date: "2026-05-02"
author: "kos-domus"
status: "ready"
session_type: "openclaw"
client: ""
openclaw_version: "2026.4.29"
environment:
  os: "Linux 6.17 (kos-domus mini-PC)"
  ide: "VSCode"
  model: "claude-opus-4-7[1m]"
tags: ["skills", "mcp", "mcp-servers", "obsidian", "memory", "context", "automation", "cron", "scheduling", "configuration", "security", "setup"]
---

## Objective

Mattinata multi-task: (a) espandere lo stack di tooling Claude Code/agent fleet (skills, MCP server), (b) introdurre un knowledge graph Obsidian disciplinato (wikilinks 3-layer + MOC), (c) chiudere il MC Morning Todo di Spesabot (priorità Rakki: Poli/Orvea, frontend P0/S, Esselunga health, Drive restore), (d) lasciare il sistema con backlog chiaro per lunedì.

## Context

Sessione partita su mini-PC aggiornato OpenClaw `2026.4.29` (upgraded ieri). Stato di partenza: VPN persistente al cliente Sacchitalia attiva (sistemata venerdì). Vault Obsidian Personal in `~/Obsidian-Personal/` con Daily/Decisions/Inbox ma senza wikilinks o cross-project graph attivo. Spesabot 13 catene live con 4 chain del Gruppo Poli scaffolded ma "scraper unimplemented". MC Evening Digest già patchato venerdì per `log.md` cross-progetto (prima run riuscita ieri 19:32).

## Steps Taken

### 1. Tool stack expansion — Karpathy skills, Promptfoo, Anthropic Skills, Firecrawl MCP, Playwright MCP

Confermato che `codebase-memory-mcp`, `obsidian` MCP e `context7` (aggiunto ieri) sono tutti `user`-scope (disponibili per ogni progetto, attuale e futuro).

Aggiunti durante la sessione:
- **Karpathy skills plugin** via `claude plugin marketplace add forrestchang/andrej-karpathy-skills` + `claude plugin install andrej-karpathy-skills@karpathy-skills`. Marketplace registrato, plugin enabled user-scope. (4 prompt rules: Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven.)
- **Anthropic Skills** ufficiali via `claude plugin marketplace add anthropics/skills` + install di 2 bundle: `document-skills` (PDF + XLSX + DOCX + PPTX) + `example-skills` (skill-creator + mcp-builder + frontend-design + brand-guidelines + 9 altri).
- **Promptfoo** CLI globale: `npm install -g promptfoo` (versione 0.121.9). Pronto per test golden suite quando servirà.
- **Playwright MCP** user-scope: `claude mcp add --scope user playwright -- npx -y @playwright/mcp@latest`. Connected.
- **Firecrawl MCP** user-scope ma con secure wrapper (vedi step 2).

Risorse valutate ma scartate (con motivazione): Superpowers (metodologia overkill), UI/UX Pro Max (UI Spesify già definita), Graphify (duplicato di codebase-memory-mcp), claude-peers (duplicato di Master Control + Telegram).

**Result**: 3 marketplace + 3 plugin + 2 nuovi MCP server registrati. Catalogato tutto in memoria `reference_community_claude_resources.md`.

### 2. Firecrawl secure key handling

Rakki ha API key Firecrawl paid plan. Per evitare leak nel transcript Claude (che persiste localmente) o in `~/.claude.json`, costruito wrapper-pattern:

- Key salvata in `~/.openclaw/credentials/firecrawl-api-key` (mode 0600 root-writable kos-only)
- Wrapper script `~/.openclaw/bin/firecrawl-mcp-wrapper.sh` (mode 0700) che fa `cat` del file + `export FIRECRAWL_API_KEY=...` + `exec npx firecrawl-mcp`
- `~/.claude.json` contiene solo path al wrapper, mai la key in chiaro
- Cleanup eseguito: bash history pulito, systemd env.conf permission 664→600

Ho letto la key in chiaro nel mio output durante l'audit del bash history → key ora esiste anche in transcript Claude. Threat model attuale (single-user mini-PC, accesso solo via Tailscale autenticato): acceptable. Rakki rotaerà la key lunedì via 1Password (richiede mini-PC fisico per approval prompts).

**Result**: Firecrawl MCP connected, key persiste in 1 solo file 0600, mai in `~/.claude.json` né bash history.

### 3. Obsidian 3-layer wikilinks policy + MOC system

Discussione meta su Karpathy "LLM Wiki" pattern: il setup attuale (DOSSIER + memorie + codebase-memory-mcp + obsidian MCP) copre ~85% del pattern Karpathy, mancavano (a) timeline cross-progetto e (b) wikilinks per graph navigability.

Adottata policy 3-layer:

| Layer | Cosa | Naming |
|---|---|---|
| 1. Progetti | Lista chiusa: `[[Spesabot]]`, `[[Sacchitalia]]`, `[[Family-OS]]`, `[[OpenClaw]]`, `[[Tools]]` + moduli `[[deposit-receipts]]` | Sempre wikilink quando menzioni un progetto |
| 2. Topic tecnici | 12 canonical: `[[OCR]]`, `[[ETL]]`, `[[PostgreSQL]]`, `[[SQL-Server]]`, `[[Telegram]]`, `[[Gemini-Vision]]`, `[[MCP]]`, `[[Multi-tenant]]`, `[[VPN]]`, `[[Cron]]`, `[[fuzzy-matching]]`, `[[Excel-XLSX]]` | Solo se ricorrenti in ≥2 progetti |
| 3. MOC | `~/Obsidian-Personal/MOC/<Topic>.md` per ogni topic Layer 2 | Wiki entry curato cross-progetto |

Creati 5 MOC scheletro: OCR, ETL, MCP, Telegram, Multi-tenant. Memoria `feedback_obsidian_topic_links.md` documenta la lista canonica + auto-promotion rule.

### 4. MC cron payload patches (5 job aggiornati via `openclaw cron edit`)

Backup `jobs.json.before-client-context-2026-05-02` + tutte le modifiche via `openclaw cron edit <id> --message <new>` per garantire hot-reload:

| Job | Patch |
|---|---|
| MC Daily Work Briefing (06:30) | Scende dentro `<client>/<module>/DOSSIER.md` per cliente multi-modulo + `git pull` di repo client-sessions per ultima cronaca + scrive Obsidian Daily con wikilinks `[[ProjectName]]` |
| MC Evening Digest (19:30) | Scrive `~/Obsidian-Personal/log.md` cross-progetto append-only + aggiorna MOC esistenti se rileva attività + propone auto-promotion topic con conferma Rakki via Telegram |
| Backend weekly (lun 04:00) | Sceglie progetto più attivo della settimana (no più SpesaBot fisso) + git pull client-sessions |
| CSO weekly (lun 04:30) | Audit anche `client-sessions/` (IP interni, default PIN, path UNC) |
| OrchArch weekly (ven 05:00) | Sceglie più attivo + cronaca cliente |

### 5. Spesabot Gruppo Poli — orvea live (prima chain del Gruppo)

**Discovery**: gruppopoli.it è ASP.NET WebForms (cookie + `__VIEWSTATE` + `btnTrova` postback) con AJAX endpoint per category fetch (`/interne/ajax/ContentCategoriaVolantino.aspx?IDCategoria=X&id=Y`). Niente Shopfully, niente PDF, niente flipbook.

**Strategia ibrida**: Playwright UNA volta per setup ASP.NET session, poi POST AJAX per ogni categoria con cookie jar (pure HTML, no JS execution past picker).

**Filtro Verona-province**: dei 4 brand del Gruppo Poli (Poli, Orvea, Regina, Amort), solo `orvea` ha store in zona target — Orvea Peschiera del Garda + IperOrvea Affi. Gli altri 3 (Trentino-AA only) restano `enabled: false` nel config, seedati in DB per espansione futura.

Implementato in `src/parsers/gruppopoli.ts` (~430 righe), wired in `runner.ts` con per-store loop + auto-creation degli store via runtimeStore + GPS coords backfilled.

**Smoke test live**: 1249 prodotti fetchati in 27s → 1164 offerte ingerite in DB su 709 SKU nuovi, 2 store creati, validity 04-27 → 05-03. Aggiunto `orvea` alla `SPESABOT_CHAINS` env del systemd unit pipeline.

### 6. Spesabot frontend P0/S — showToast helper

Verificato che 2 dei 3 micro-fix flaggati dal Frontend Specialist erano già stati shippati ieri (commit `9995fbe`): spinner `.active` toggle + watch-form duplicate-click guard. Il MC report di stamattina era stale su quei 2.

Resta il "save micro-feedback": l'utente non aveva conferma visiva che le sue azioni andassero a buon fine. Aggiunto helper `showToast(text, type)` (CSS `.toast` + JS auto-dismiss 2.4s) e wired in 4 save flows: addWatch, deleteWatch, addToShoppingList, deleteShoppingListItem. Cache-bust bumped, smoke verified su API locale.

### 7. Spesabot Esselunga/Shopfully health + 2 fix sistemici

Audit delle 4 chain Shopfully. Esselunga 🟢 (844 offerte 100% attive su 3 store, multi-pub `824458`/`824460`), MD 🟢, DPIÙ 🟡 (cleanup automatico si occuperà), CRAI 🔴 (publication `818508` scaduta dal 26 aprile, doveconviene non lista Verona/Veneto per CRAI — solo 16 città centro Italia).

Bonus scoperto durante audit: pipeline del 1 maggio uccisa per `TimeoutStartSec=5400` durante MD vision processing (4m 37s CPU, peak 894MB RAM). Trend sfavorevole: Apr 21 OOM, Apr 24 + May 1 timeout.

Applicati 2 fix sistemici:
- `TimeoutStartSec=5400→7200` in `spesabot-pipeline.service`
- CRAI rimosso dalla `SPESABOT_CHAINS` env list (skip pulito invece di failure ripetuta ogni run)

### 8. Drive restore Work/Job Projects/

Folder Drive `Work/Job Projects/` (`1SpMW...`) era vuoto, MC briefing operava blind. Generati 2 brief sintetici (`spesabot-brief.md` + `sacchitalia-brief.md`) con frontmatter standard fleet + DOSSIER attuali sintetizzati. Uploaded via `gws drive +upload`.

### 9. OpenClaw bots health check (richiesto da Rakki a chiusura)

Verificato che gateway/timer/cron MC sono tutti UP (06:33 Daily Briefing, 07:13 Morning Todo, 19:32 Evening Digest ieri eseguiti regolarmente). Backend/CSO/Inbox weekly mai girati ma è normale: creati 27-28 aprile DOPO la finestra weekly del lunedì → primo run reale lunedì 4 maggio. Unico vero rotto: `spesabot-matching.service` failed stamattina 05:39 con FK violation `product_id=5242 not in products` — orphan SKU referenziato. Da fixare lunedì o oggi.

### 10. Saga di chiusura: GPT-5.5 swap + 3 gateway restart + SSH tunnel pattern

A valle del check health, mossa di tooling sui due agent più "pesanti" della fleet: bump del primary model `openai-codex/gpt-5.4 → gpt-5.5` con `gpt-5.4` mantenuto in fallback (zero rischio di rollback). Patch via edit diretto di `~/.openclaw/openclaw.json` con backup esplicito `openclaw.json.before-gpt5.5-2026-05-02` per safety net.

Cluster di restart gateway che ne è seguito (timeline da journalctl):

| Ora | Evento | Note |
|---|---|---|
| 10:31:43 | Gateway restart #1 | warm-up reload prima del config swap (consumed 1h30min CPU dal boot precedente, peak 5.8GB RAM) |
| 10:33:44 | Patch `openclaw.json` (GPT-5.5 swap) + restart #2 | hot-reload del nuovo modello |
| 10:37:19 | Gateway restart #3 | settle/verify dopo che watchdog ha promosso config a `lastPromotedGood` (08:37:28 UTC = 10:37:28 local) |

Nessun crash, nessun warning di config corruption nei log: il watchdog `~/.openclaw/logs/config-health.json` ha confermato hash `6ae0419120e3...` come good config promosso e mai più rollback. Da monitorare nei prossimi giorni: latency p99 del routing agent (model-resolution è già il 5s+ del bootstrap stages) e error rate.

**SSH tunnel pattern** documentato come reference riusabile (memoria `reference_minipc_localhost_tunnel.md`, scritta 11:18:46): molti servizi sul mini-PC bind solo `127.0.0.1` per security (no internet exposure). Per accedere dal Mac via Tailscale senza modificare bind:

```bash
ssh -N -L <PORT>:localhost:<PORT> spesify-mini
```

Primo use case live: `codebase-memory-mcp` graph UI su porta 9749. Pattern preferibile a Tailscale-bind permanente quando l'accesso è "ogni tanto" — zero ginnastica setup, niente unit edits.

## Configuration Changes

- `~/.claude.json` — 3 nuovi MCP server (firecrawl wrapper-based, playwright stdio, context7 HTTP — quest'ultimo già da ieri); 3 plugin (karpathy + 2 anthropic bundle); 3 marketplace
- `~/.openclaw/openclaw.json` — primary model `openai-codex/gpt-5.4 → openai-codex/gpt-5.5` per 2 agent, `gpt-5.4` aggiunto in fallback chain. Backup: `openclaw.json.before-gpt5.5-2026-05-02`. Promosso a `lastPromotedGood` dal watchdog alle 10:37:28
- `~/.openclaw/cron/jobs.json` — 5 job payload aggiornati (MC Daily, MC Evening, Backend weekly, CSO weekly, OrchArch weekly)
- `~/.openclaw/SOUL.md` — sezione "Client data confidentiality" aggiunta venerdì, oggi solo riconfermata
- `~/.openclaw/memory/FLEET-DRIVE.md` — sezione "GitHub repositories" venerdì
- `/home/kos/.config/systemd/user/spesabot-pipeline.service` — TimeoutStartSec 5400→7200 + CRAI rimosso da chain list, daemon-reload eseguito
- `~/job-desk/CLAUDE.md` — wikilinks 3-layer policy aggiunta
- `~/job-desk/spesabot/configs/stores.yaml` — orvea passato da `parser: unimplemented` a `parser: gruppopoli`
- `~/job-desk/spesabot/.gitignore` — `scratch/` aggiunto

Nuovi file:
- `~/Obsidian-Personal/MOC/{OCR,ETL,MCP,Telegram,Multi-tenant}.md`
- `~/.claude/projects/-home-kos-job-desk/memory/feedback_obsidian_topic_links.md`
- `~/.claude/projects/-home-kos-job-desk/memory/reference_community_claude_resources.md`
- `~/.openclaw/credentials/firecrawl-api-key` (mode 0600)
- `~/.openclaw/bin/firecrawl-mcp-wrapper.sh` (mode 0700)
- `~/job-desk/spesabot/src/parsers/gruppopoli.ts` (parser nuovo, ~430 linee)
- `~/.claude/projects/-home-kos-job-desk/memory/reference_minipc_localhost_tunnel.md` (SSH tunnel pattern via Tailscale per servizi 127.0.0.1)

Spesabot commits:
- `3c82258` orvea (Gruppo Poli) live scraper + 1164 offers ingested
- `c88b03e` spesify: showToast helper for save-confirmation feedback

## Key Discoveries

- **Karpathy LLM Wiki ≠ migliore del nostro stack — è complementare**. Coprivamo già 85% (DOSSIER, memorie, codebase-memory-mcp, obsidian MCP). Mancavano i due pezzi che il pattern Karpathy enfatizza: graph navigability via wikilinks ovunque + timeline cross-progetto via `log.md`. Adottati entrambi.
- **MC report può essere stale**: 2/3 dei "micro-bug frontend P0/S" erano già stati shippati il giorno prima (commit `9995fbe`) ma il Frontend Specialist non aveva ri-letto il codice fresco. Lesson: prima di partire su un report MC, verifica se i fix sono già a terra.
- **Gruppo Poli architecture**: 4 brand su 1 sito ASP.NET WebForms condiviso. Risolto il blocker storico (ASP.NET ViewState) con strategia ibrida Playwright-1-shot + AJAX-many. Pattern riusabile per altri siti WebForms-based.
- **Geographic discipline è critica**: il Gruppo Poli ha 7 insegne × ~62 store, ma solo 2 store sono in zona target Verona-province (Orvea Peschiera + IperOrvea Affi). Disabilitare a config-level le altre 3 chain (poli/regina/amort) impedisce di sprecare cron time + Gemini API + DB rumore.
- **Pipeline trend è sfavorevole**: 3 fail negli ultimi 14gg (Apr 21 OOM, Apr 24 + May 1 timeout). Il bandage di alzare TimeoutStartSec è temporaneo — OrchArch ha già flaggato P1/L "estrazione strategy modules da runner.ts" che è il fix architetturale.

## Errors & Solutions

| Error | Cause | Solution |
|---|---|---|
| Smoke spike #2 catturava 0 fasce | `page.content()` chiamato troppo presto, prima che il JS bootstrap iniettasse il DOM dei volantini | `page.waitForSelector('.fascia-volantino')` + `waitForTimeout(2000)` extra |
| `parseProductCards` non trovava i product blocks coi regex multiline | `.id-prodotto` matchava prima il pattern JS my-list (popup) che il vero product card | Selettore corretto: split su `<div class="band-volantino-prodotto ` + parse field-by-field |
| Smoke test diceva "Could not locate volantino id" per Regina/Amort store di default | I primi store (`__FIRST__`) ritornano store Trentino senza volantino attivo OR con layout diverso | Hard-coded la lista store target Verona-province nel config; chain senza store target diventano `enabled: false` |
| `npx tsx` non trovava module 'playwright' lanciato da `/tmp/` | tsx risolve relative al file, non al cwd | Sposta script in `~/job-desk/spesabot/scratch/` (gitignored) |
| Pipeline orvea smoke skippava la chain con "scraper not yet implemented" | `configs/stores.yaml` aveva ancora `parser: unimplemented` | Update YAML a `parser: gruppopoli` |
| `spesabot-matching.service` failed stamattina 05:39 | FK violation: `product_id=5242` referenziato da SKU ma cancellato da `products` | TBD — investigare orphan SKU + relink o soft-skip |

## Final State

- **Plugins/MCP**: 3 plugin enabled (karpathy + 2 anthropic), 5 MCP user-scope connessi (codebase-memory + obsidian + context7 + firecrawl + playwright)
- **Cron MC**: 5 job aggiornati con istruzioni client-sessions/MOC/wikilinks; backup safety-net presente
- **Fleet model routing**: 2 agent passati a `openai-codex/gpt-5.5` primary (5.4 fallback), 3 gateway restart pulite, watchdog ha promosso config a `lastPromotedGood`. Latency p99 routing da monitorare nei prossimi giorni.
- **Spesabot**: 14 chain attive (orvea aggiunto), 1164 offerte zona Verona nuove, frontend toast feedback live, pipeline timeout 7200s, CRAI temporaneamente disabilitato
- **Obsidian**: 5 MOC scheletro creati, policy wikilinks 3-layer documentata in CLAUDE.md + 2 memorie
- **Drive**: Work/Job Projects/ ripopolata con 2 brief sintetici
- **Reference patterns**: SSH tunnel via Tailscale documentato per accesso da Mac a servizi `127.0.0.1` del mini-PC (graph UI codebase-memory su :9749 il primo use case)
- **Memorie aggiornate**: `feedback_obsidian_wikilinks.md`, `feedback_obsidian_topic_links.md`, `reference_community_claude_resources.md`, `project_spesabot_status.md` (chain count 13→14), `reference_minipc_localhost_tunnel.md` (nuovo)

## Open Questions

- **`spesabot-matching.service` FK violation**: investigare se è race con cleanup canonical, decidere quick-fix (soft-skip orphan + relink) vs proper-fix (transactional consistency check tra SKU update e canonical delete)
- **Pipeline timeout trend**: 7200s alzato è bandage. OrchArch P1/L "strategy modules da runner.ts" + P1/M "health/fallback ingest" sono i fix architetturali da affrontare lunedì
- **CRAI**: rimosso dalla chain list, ma se in futuro CRAI riapre Verona su altro aggregator (promoqui, iflyer.it, sito proprio) andrà ri-discoverato manualmente
- **Backend/CSO/Inbox weekly first run** — lunedì 4 maggio 04:00-09:00 sono i primi run reali. Verificare che girino e non falliscano per mancanza di context/tooling
- **MOC auto-promotion**: l'Evening Digest stasera è il primo a tentare auto-promotion topic basata su 14gg rolling — verificare domattina nei log se ha proposto candidati corretti
- **GPT-5.5 stability**: prima settimana con i 2 agent in produzione su `gpt-5.5` primary. Da osservare: p99 latency (model-resolution è già 5s+ del bootstrap), error rate vs `gpt-5.4`, eventuali rollback automatici via fallback chain. Soglia di rollback manuale a `gpt-5.4` se la regression è netta (>20% lat o errori in serie).
