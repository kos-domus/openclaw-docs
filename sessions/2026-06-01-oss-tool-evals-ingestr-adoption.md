---
title: "Valutazione portfolio OSS (Co-CEO, 20 progetti) + adozione ingestr con vendor-gate CSO + check pliny.gg"
date: "2026-06-01"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "security", "cli", "automation", "agent-sdk", "troubleshooting"]
session_type: "openclaw"
openclaw_version: ""
environment:
  os: "Linux (mini-PC)"
  ide: "Claude Code (VSCode extension)"
  model: "claude-opus-4-8"
---

## Objective
Far valutare al Co-CEO due liste di progetti OSS / risorse (~20 item totali) per decidere cosa vale tempo/€; poi pilotare e mettere in sicurezza ciò che passa; infine valutare pliny.gg per prompt-engineering.

## Steps Taken
### 1. Due workflow Co-CEO (fan-out ricerca → sintesi strategica)
Workflow A: 11 OSS + blog agent-architecture + risorse apprendimento Claude Code. Workflow B: 9 repo di un post virale ("10 repos so good they shouldn't be free"). Ogni progetto ricercato in parallelo (WebFetch + maturità/licenza/fit) → sintesi Co-CEO con verdetti prioritizzati.
**Result**: su 20 progetti, **solo 2 meritano tempo di prova** (ingestr, Printing Press). Il post virale = rumore (trading/media-gen off-topic). Risorse apprendimento: utente già oltre il 90%, leggere solo "Routines" docs + 1-2 moduli eval.

### 2. Pilot ingestr (CLI Go data ingestion)
Install `uv tool install ingestr` (7s, isolato). Test self-contained sqlite→duckdb + **delta-load incrementale** (`--incremental-strategy merge --incremental-key`): aggiunta 1 riga → dest sincronizzata senza re-copy full. UX pulita (auto-PK, staging, metriche).
**Result**: ingestr passa — sostituisce glue-code ETL per flussi source-strutturata→DB (es. SQL Server → Postgres staging). NON per web-scraping.

### 3. Vendor-gate CSO per uso su dati cliente → GO-with-conditions
Il CSO ha corretto una premessa: **ingestr v1.x è un BINARIO Go**, non Python/dlt. Conseguenze:
- La `~/.dlt/config.toml` (telemetry dlt) è **INERTE** sul binario Go. La telemetry vera è propria (Go → RudderStack/mixpanel: versione/OS/comando, NO dati/schema) → kill con env `INGESTR_DISABLE_TELEMETRY=true`.
- `trivy fs` sul venv = falso negativo (venv vuoto, è un wrapper). `trivy rootfs` sul binario → 19 CVE (2 CRITICAL: driver Postgres + Go-TLS; 17 HIGH DoS-class, basso rischio reale in topology a 2-endpoint-fidati).
- Licenza FSL-1.1: ok per uso operativo (non ridistribuito nel deliverable).
**Result**: GO con 4 condizioni — C1 telemetry off (env, non la config dlt), C2 source read-only, C3 pinned+monitor (re-scan `trivy rootfs` su bump), C4 secret via env (no inline). Creata skill wrapper `ingestr-data-sync` + vendor-tool register.

### 4. Printing Press → parkeggiato
Genera CLI Go + skill Claude Code + skill OpenClaw + MCP da un prompt (fit tri-runtime). Blocker: richiede Go 1.26.3+ (presente 1.22.2 → toolchain auto-download) + installa skill 3rd-party nell'ambiente live + **è codegen → gate CSO**. Pilot pronto, richiede OK esplicito.

### 5. pliny.gg
Sito di Pliny the Prompter (jailbreaker): L1B3RT4S, refusal-removal, prompt-injection, system-prompt leak. **Zero alpha per prompt-engineering legittimo** (è adversarial, direzione opposta a costruire sistemi affidabili). Valore solo **difensivo**: input threat-intelligence per hardening anti prompt-injection.

## Key Discoveries
- **Co-CEO come filtro ruthless**: una lista "tutta interessante" è una shiny-object trap; il valore reale è quasi sempre 1-2 item che muovono il P&L. Regola: max 1 pilot/volta, time-box 3h, rule-of-three anche per i tool, Watch ≠ todo.
- **Gotcha telemetry tool "Python" riscritti in Go**: i kill-switch documentati per la versione Python (config file) possono essere INERTI sul binario Go — verificare il meccanismo reale (env var) con string-analysis, non fidarsi della config legacy.
- **Pattern "GO archiviato" + vendor-tool register**: un tool di terze parti per dati cliente non è "sicuro" perché gira pulito localmente — serve gate CSO (telemetry/supply-chain/licenza) + register con baseline CVE + condizioni d'uso, re-scan su ogni bump.
- **Tool esterno ≠ embed**: usarlo come CLI operativo (su nostra infra) tiene il supply-chain rischio fuori dal codice/deliverable cliente. Mai bundlare un binario di terze parti in un deliverable.

## Final State
ingestr adottato + messo in sicurezza (skill + register + memoria). Printing Press parked con piano. pliny archiviato come fonte difensiva. Portfolio: tutto il resto archiviato (off-topic o lettura).

## Open Questions
- Printing Press pilot (Go toolchain + OK + CSO sul codegen).
- Re-eval di alcuni repo trading sulla qualità (progetto futuro dedicato).
