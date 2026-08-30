## 2026-08-30 — Daily KB Processing (automated)

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

