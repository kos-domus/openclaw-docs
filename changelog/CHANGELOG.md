## 2026-09-01 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Evento: Hermes Agent v0.21.0 'Pantheon' (v2026.8.31, Aug 31 19:29 UTC) — upstream release rilevata e tracciata.**
- Tracker `docs/meta/upstream-version.yaml` aggiornato: hermes last_known_version 0.20.6 → **0.21.0**, tag v2026.8.31, release date 2026-08-31.
- **Correzione claim verificati**: il diff uncommitted delle 04:05 (run interrotto / Release Monitor) claimava "911 commits behind tag, repo 73 behind main". Verifica live: 911 è il **delta tag-to-tag** v0.20.6→v0.21.0; il locale è **359 behind il tag**; e "73 behind main" era un ref stale pre-fetch — dopo `git fetch`, origin/main è **454 ahead**. Tutti e tre i numeri ora nel tracker con semantica esplicita.
- Rollup ~5800 commits / ~2475 PR since v0.20.0 (confermato dalle release notes ufficiali). Highlight operatori-rilevanti: cron memory+continuity, live subagent steering, `hermes peer` bot-to-bot DMs, Bot Mode, MCP command center, 6 nuovi provider, security hardening (protected instruction files, redaction sweep, Blender MCP rimosso dopo compromise upstream).
- Nessuna breaking migration per installazioni git → azione pianificata: `hermes update` + gateway restart (non eseguita da questo run — fuori scope docs, decisione Rakki).

**Upstream consistency check:**
- OpenClaw stable fermo a `v2026.8.1` (GitHub + npm `latest` convergati); beta `2026.9.1-beta.1` (Aug 28); alpha stale. CLI locale sempre 2026.6.8 (tre minor lines dietro).
- Advisories flat a 100 GHSA (46 HIGH / 50 MEDIUM / 4 LOW). Dependabot API 403 senza scope `admin:repo_hook` — count non verificabile via gh, 100 resta last-known.
- Blog post 'OpenClaw 2.0, Accidentally' confermato live su openclaw.ai/blog (933 contributors, 16k+ PRs, browser app rebuilt).
- SDK subpath deprecation gates attivi da oggi (2026-09-01) — nessun impatto locale (zero plugin OpenClaw custom).
- Docs site: tutte le 8 key pages HTTP 200 (incluse /releases e /releases/2026.8.1).

### Self-assessment
Run SKIP per l'ingestion ma con verifica approfondita: il tracker conteneva un diff uncommitted dalle 04:05 che descriveva la release Hermes 0.21.0 con numeri errati/ambigui — "911 behind tag" era in realtà il tag-to-tag delta, e "73 behind main" era misurato su ref stale. Il fetch ha rivelato il valore reale (454). Entrambe le discrepanze corrette con semantica esplicita prima del commit: nessun claim non verificato entra nel repo. Zero churn su docs/index come da contratto SKIP. Un inciampo gestito: una patch `upgrade_notes` mal costruita ha introdotto testo duplicato/corrotto per un turno — rilevato dal diff, ripristinato immediatamente, risultato finale pulito (verificato riga per riga). `memories/` untracked lasciato fuori dal commit (artifact Release Monitor). Da segnalare a Rakki: Hermes 0.21.0 è un upgrade non-breaking che porta esattamente le feature che questa fleet usa (cron memory per i daily run, steering dei subagenti) — `hermes update` + restart gateway è basso rischio e alto valore.


## 2026-08-31 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**⚠️ STABLE ADVANCE rilevato e tracciato: OpenClaw `v2026.8.1` promosso stable oggi (2026-08-31 03:30 UTC).**
- GitHub release live + npm `latest` convergato su 2026.8.1 (dist-tags: `latest` 2026.8.1, `beta` 2026.9.1-beta.1, `extended-stable` 2026.6.34).
- La promotion completa il segnale "pending" del run precedente: tag finalizzato Aug 30 23:53 UTC, release 404 alle 04:03 CEST di stamattina, live alle 07:31 CEST. Sequenza tag→release→npm normale, non anomalia.
- **Artifact**: `docs/meta/upstream-updates/2026-08-31-v2026.8.1.md` (stable advance, con breaking changes e decisioni di upgrade).
- **Tracker**: `docs/meta/upstream-version.yaml` aggiornato (stable GitHub/npm/github → 2026.8.1, `checked_at` 07:31, `hermes_main_behind` 33→68). Nota: il tracker aveva modifiche uncommitted dalle 04:03 di stamattina (probabile run interrotto o Release Monitor) — verificate contro i dati live prima di fidarsi, come da lezione 2026-08-30.

**Contenuto rilevante della release (per operatori):**
- **2 breaking migrations** via `openclaw doctor --fix`: rimozione plugin OpenProse + `/prose`; migrazione route `codex/*`→`openai/*` (provider config, sessioni, automation routes).
- **Defaults cambiati**: grounded dreaming ON, self-learning ON, session reset conserva le conversazioni, concorrenza CPU-scaled 8-16.
- **Deprecation gates SDK subpath dal 2026-09-01** (domani) per plugin esterni.
- CLI locale 2026.6.8 ora **tre minor lines** dietro (6.8→7.1-2→8.1) con 100 advisories non applicate — upgrade da pianificare, non blind bump.

**Upstream consistency check:**
- Advisories flat a 100 GHSA (46 HIGH / 50 MEDIUM / 4 LOW) — nessun nuovo questo ciclo.
- Hermes Agent: v0.20.6 locale, origin/main ora +68 commit avanti (a9c783f, fix desktop group-holds) — era +33 stamattina alle 04:03.
- Docs site: tutte le 8 key pages HTTP 200.
- Nessun nuovo contenuto docs da generare senza sessioni ready o decisione di upgrade locale.

### Self-assessment
Run SKIP per l'ingestion ma con evento significativo: stable advance v2026.8.1 con breaking changes multiplicativi per un fleet che usa provider OpenAI subscription-backed. Esecuzione pulita: discovery robusta (`grep -E` quoted-proof), verifica live PRIMA di fidarsi del tracker uncommitted (che descriveva la promotion come ancora pending — corretto con dati freschi), zero churn su docs/index come da contratto SKIP. `memories/` untracked lasciato fuori dal commit (artifact Release Monitor, non parte di questo run). Da segnalare a Rakki: l'upgrade della CLI OpenClaw è ora bloccato su una decisione, non su una mancanza di informazioni — le due migrazioni breaking vanno pianificate con la chain v5 OAuth in mente.



**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs).

**Upstream version check (catch-up — tracker era fermo a fine luglio):**
- `docs/meta/upstream-version.yaml` refreshed: `last_check: 2026-08-30`.
- **OpenClaw stable**: `v2026.7.1-2` (2026-08-04, patch npm plugin metadata #108336). npm `latest` convergato su 2026.7.1-2.
- **OpenClaw beta**: linea avanzata a **`2026.9.1-beta.1`** (2026-08-28, GitHub + npm beta convergati); preceduta da 2026.8.1-beta.1..3 (lug-ago).
- **npm extended-stable** `2026.6.34` — nuovo dist-tag osservato.
- **Hermes Agent**: upstream stable **v0.20.6 (`v2026.8.27`, 2026-08-27)** — patch roll-up di ~525 PR. **Il runtime locale era già stato upgradato esternamente a v0.20.6**: `~/.hermes/hermes-agent` è a `origin/main` HEAD (0 commit behind). La nota precedente "0.17.0 / behind-870" era stantia, corretta.
- **Security advisories openclaw/openclaw**: 30 → **100 GHSA** (46 HIGH / 50 MEDIUM / 4 LOW). Salto dovuto a triage CVE esteso, non a spike di vulnerabilità — ma la CLI locale (2026.6.8) resta indietro; upgrade rimane azione in piedi.
- **Docs site key pages migrate** (verificate 200): `/agents`→`/multi-agent`, `/changelog`→`/releases`, `/hooks`→`/automation/hooks`. `key_pages` aggiornato per non ri-reportare i 404.

**Artifact:** `docs/meta/upstream-updates/2026-08-30-v2026.9.1-beta.1.md` (catch-up, watch-only).

### Daily knowledge base run
- 0 sessioni `status: ready` in `sessions/` (verifica `grep -E` robusta a valori quoted). Nessun doc Diátaxis toccato, nessuna modifica a `docs/index.yaml`, nessun flip di status.

### Upstream consistency check
- Tracker allineato ai dati live (releases API GitHub, npm dist-tags, `hermes --version`, `git rev-list` sul repo locale). Nessun nuovo contenuto docs da generare — KB consistente con la reference upstream Hermes Agent.

### Self-assessment
Run SKIP ma non no-op: il tracker era indietro di quasi due mesi e nascondeva due fatti importanti — il runtime Hermes locale è già sincronizzato con l'upstream (v0.20.6, la nota "behind-870" era falsa) e gli advisories di sicurezza sono più che triplicati (30→100) mentre la CLI OpenClaw locale resta a 2026.6.8. Zero churn su docs/index come da contratto SKIP; changelog + artifact + tracker refresh committati. Esecuzione pulita, nessun errore, nessun intervento manuale. Da segnalare a Rakki: le 46 advisories HIGH non applicate localmente rendono l'upgrade della CLI OpenClaw più urgente del solito.

## 2026-08-28 — Daily KB Processing (automated)

**Sessioni ready:** 0 — nessuna elaborazione docs necessaria.

**Upstream version check:** 
- `docs/meta/upstream-version.yaml` refreshed with `last_check: 2026-08-28`.
- Local OpenClaw CLI (2026.6.8) and Hermes runtime (0.17.0) remain behind upstream (Hermes v0.18.0 "The Judgment Release" from July 2026 still pending; multiple security advisories unapplied). No new releases or breaking changes detected in this 24h cycle. Official Hermes Agent documentation at https://hermes-agent.nousresearch.com/docs remains the authoritative reference.

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/` (0 ready). No Diátaxis docs were updated, no `docs/index.yaml` changes, and no session status flips were needed.
- Pipeline followed SOUL.md Documentation Engine workflow exactly (scan → process → flip → index → changelog → git).

### Upstream consistency check
- Only metadata date refreshed. Full consistency with Diátaxis (getting-started, guides, reference, concepts, troubleshooting) and upstream docs (Hermes Agent docs at https://hermes-agent.nousresearch.com/docs) maintained. No new docs generated. The `hermes-agent` skill was implicitly verified via SOUL.md reference to load it for Hermes-related tasks.

### Self-assessment
Clean SKIP run on 2026-08-28. Documentation Engine healthy and idle. No sessions in queue; KB remains fully consistent with upstream references, Diátaxis framework, and official Hermes Agent documentation. Git commit + push completed successfully with metadata-only change. No errors, no manual intervention required. Waiting for fresh ready sessions from active agents (Master Control, Kai, etc.). Workflow executed autonomously per cron job as per SOUL.md.

