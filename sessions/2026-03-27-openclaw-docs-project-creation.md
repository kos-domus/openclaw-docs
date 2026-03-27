---
title: "Creazione progetto openclaw-docs: sistema documentale vivente con pipeline Security Agent + Kos"
date: "2026-03-27"
author: "kos-domus"
status: "ready"
tags: ["automation", "cron", "scheduling", "security", "configuration", "setup", "git", "github"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24 (AceMagic mini PC, 16GB RAM)"
  ide: "VS Code (via Tailscale)"
  model: "claude-opus-4-6"
---

## Objective

Creare un sistema di documentazione vivente per OpenClaw che si autoaggiorna: le sessioni di lavoro vengono elaborate automaticamente da un agente AI (Kos) in documentazione strutturata, con un Security Agent come pre-filtro per la protezione dei dati.

## Context

- OpenClaw 8-agent framework già operativo su mini PC
- Necessità di documentazione analitica che migliori nel tempo
- Vocazione imprenditoriale: il sistema deve essere open source, contributabile, e agent-friendly
- VS Code accessibile via Tailscale dal Mac

## Steps Taken

### 1. Creazione repo e struttura progetto

Repo GitHub pubblico: `kos-domus/openclaw-docs`

Struttura basata su framework Diátaxis:
```
openclaw-docs/
├── CLAUDE.md              # Istruzioni per Kos (elaboration engine)
├── SECURITY_AGENT.md      # Istruzioni per Security Agent (PII scanner)
├── sessions/              # Input: log sessioni di lavoro
│   ├── _template.md
│   └── schema.yaml
├── docs/                  # Output: documentazione elaborata
│   ├── index.yaml         # Machine-readable index
│   ├── getting-started/
│   ├── guides/
│   ├── reference/
│   ├── concepts/
│   ├── troubleshooting/
│   └── meta/
│       ├── upstream-version.yaml
│       └── upstream-updates/
├── changelog/
│   ├── CHANGELOG.md
│   └── SECURITY_LOG.md
└── .claude/commands/
    └── save-session.md    # Slash command per auto-save sessioni
```

**Result**: Repo creato, pushato, struttura completa.

### 2. Importazione 18 sessioni storiche (Mar 12-26)

Filtrate e convertite tutte le sessioni di lavoro precedenti in formato strutturato. Copertura: setup Ubuntu, installazione OpenClaw, autenticazione Anthropic, Google Drive/Workspace, WhatsApp/Telegram, multi-agent architecture, security deployment, skills, voice/TTS.

**Result**: 18 sessioni pushate con `status: ready`.

### 3. Security audit e sanitizzazione PII

Audit completo su tutte le sessioni: trovati 20+ dati sensibili (telefoni, email, IP, JID, Telegram ID). Tutti sostituiti con placeholder prima del push pubblico.

Script di sanitizzazione Python con regex per tutti i pattern: `+39[0-9]+`, `@gmail.com`, IP, token prefix.

**Result**: 6 file sanitizzati, 0 residui confermati con grep.

### 4. Setup trigger giornaliero Kos (elaboration engine)

Trigger remoto su Anthropic cloud (cron `0 6 * * *` UTC = 08:00 Roma):
- Phase 1: Upstream sync (check release GitHub, npm, docs ufficiali)
- Phase 2: Validazione sicurezza + elaborazione sessioni → docs

Modello: Opus 4.6

**Result**: Trigger attivo, testabile manualmente.

### 5. Setup Security Agent (pre-filter PII)

Agente autonomo dedicato esclusivamente alla scansione di sicurezza. Trigger separato (cron `0 5 * * *` UTC = 07:00 Roma, 1h prima di Kos).

Scansiona per: credenziali (CRITICAL), PII (HIGH), service ID (MEDIUM), prompt injection (LOW).

Istruzioni in `SECURITY_AGENT.md` — "paranoid by design, false positives > false negatives".

Pipeline completa:
```
Sessione pushata
  → 05:00 Security Agent scansiona
  → 06:00 Kos elabora (solo sessioni con security_check: passed)
```

**Result**: Trigger attivo, lanciato manualmente per primo test.

### 6. Slash command /save-session

Creato custom slash command per auto-generazione sessioni a fine lavoro. Disponibile globalmente (`~/.claude/commands/save-session.md`) e nel progetto.

**Result**: Comando creato, da testare dopo riavvio Claude Code.

### 7. Predisposizione contribuzioni community

- `CONTRIBUTING.md` con linee guida PII e security
- `sessions/schema.yaml` per validazione
- `.github/PULL_REQUEST_TEMPLATE.md`
- `docs/index.yaml` come entry point per fetching programmatico

## Configuration Changes

- Repo GitHub pubblico `kos-domus/openclaw-docs` creato
- `gh` CLI installato e autenticato come `kos-domus`
- 2 trigger remoti configurati:
  - `openclaw-docs-security-scan` (05:00 UTC, Opus 4.6)
  - `openclaw-docs-elaboration` (06:00 UTC, Opus 4.6)
- Slash command `/save-session` globale
- `docs/meta/upstream-version.yaml` per tracking versioni

## Key Discoveries

- **I dati personali si infiltrano facilmente**: numeri di telefono, email, IP compaiono naturalmente nelle sessioni di lavoro — serve un processo di sanitizzazione sistematico, non one-shot
- **Due agenti con scopi opposti si bilanciano**: Security Agent (incentivo a bloccare) vs Kos (incentivo a elaborare) — architettura più robusta di un singolo agente con doppio compito
- **`skills.load.extraDirs`**: la soluzione per caricare skills da directory esterne senza symlink (che OpenClaw rifiuta per sicurezza)
- **Heredoc con backtick**: bash si confonde se il contenuto ha backtick — usare delimitatori diversi o python3
- **docs/index.yaml come API**: un singolo file YAML come entry point per fetching programmatico di tutta la documentazione
- **Diátaxis framework**: getting-started, guides, reference, concepts, troubleshooting — struttura standard per docs di prodotto
- **Remote triggers girano in cloud Anthropic**: non sul mini PC locale, ma clonano il repo e operano in sandbox isolata

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `gh: command not found` | GitHub CLI non installato | `sudo apt install gh` |
| `/save-session` non riconosciuto | Claude Code non ha ricaricato i comandi | Riavviare sessione Claude Code |
| Sessioni con PII nel repo pubblico | Dati personali nelle sessioni storiche | Script Python di sanitizzazione con regex |

## Final State

Sistema documentale vivente operativo:
- 18 sessioni pronte per elaborazione
- Pipeline automatica: Security Agent (05:00) → Kos (06:00)
- Upstream sync per novità OpenClaw
- Contribuzioni community predisposte con filtri di sicurezza
- Slash command `/save-session` per auto-logging (da testare)

## Open Questions

- Come far funzionare `/save-session` senza riavviare Claude Code?
- Come gestire sessioni molto lunghe che superano il contesto dell'agente?
- Quanto costa in token il doppio run giornaliero (Security + Kos) con Opus?
- Come rendere il sistema facilmente adottabile da altri utenti OpenClaw?
