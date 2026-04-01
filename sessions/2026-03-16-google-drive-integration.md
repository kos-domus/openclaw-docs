---
title: "Configurazione integrazione Google Drive autonoma per OpenClaw"
date: "2026-03-16"
author: "kos-domus"
status: "processed"
tags: ["configuration", "mcp", "mcp-servers", "automation", "remote"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (Agimagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Configurare OpenClaw per lavorare in autonomia su Google Drive: creare, modificare, cancellare e condividere file (presentazioni, documenti, Google Sheets) usando un account Google dedicato su un mini PC headless Ubuntu.

## Context

- OpenClaw gira su un Agimagic mini PC con 16GB RAM e Ubuntu
- Il bot ha già accesso diretto e privilegiato alla macchina
- Serve un account Google dedicato (NON personale) per sicurezza
- La macchina è headless → il flusso OAuth richiede tunnel SSH

## Steps Taken

### 1. Identificazione delle opzioni di integrazione

Due percorsi principali individuati:

**Opzione A — google-workspace-mcp (setup rapido)**
- Usa `@presto-ai/google-workspace-mcp`
- Accesso diretto a Gmail, Calendar, Drive, Docs, Sheets, Slides
- Non richiede progetto su Google Cloud Console
- Setup via playbooks:

```bash
npx playbooks add skill openclaw/skills --skill google-workspace-mcp
```

**Opzione B — gogcli / GOG skill (più flessibile, consigliato per produzione)**
- CLI unificato per Gmail, Calendar, Drive, Contacts, Tasks, Sheets, Docs
- Richiede abilitazione API su Google Cloud Console
- Più controllo granulare sulle permission

**Result**: Scelta Opzione B (gogcli) per maggiore flessibilità e controllo in produzione.

### 2. Setup Google Cloud Console

Passaggi richiesti:
1. Entrare nel progetto Google Cloud
2. APIs & Services → Library
3. Abilitare: Google Drive API, Gmail API, Google Calendar API
4. Configurare OAuth consent screen
5. Creare credenziali OAuth 2.0
6. Scaricare `client_secret.json`

```bash
# Trasferire credenziali sul mini PC
scp client_secret.json utente@IP_AGIMAGIC:~/.openclaw/credentials/
```

**Result**: Guida preparata per il bot, da eseguire step by step.

### 3. Autenticazione OAuth su server headless

Il punto critico: su Ubuntu headless il browser non può aprirsi. Soluzione: tunnel SSH.

```bash
# Dal PC locale, creare il tunnel
ssh -L 8080:localhost:8080 utente@IP_AGIMAGIC
```

Poi completare il flusso OAuth su `http://localhost:8080` dal browser locale.
Le credenziali vengono salvate in `~/.config/google-workspace-mcp/` localmente.

**Result**: Pattern documentato per OAuth headless via SSH tunnel.

### 4. Comandi Drive disponibili dopo autenticazione

```bash
# Listare file
mcporter call --server google-workspace --tool "drive.list" pageSize=20

# Cercare file
mcporter call --server google-workspace --tool "drive.search" q="name contains 'report'"

# Creare una cartella
mcporter call --server google-workspace --tool "drive.createFolder" name="Progetti 2026"

# Caricare un file
mcporter call --server google-workspace --tool "drive.upload" name="report.pdf" path="./report.pdf"

# Condividere un file
mcporter call --server google-workspace --tool "drive.share" fileId="<id>" email="collaborator@example.com" role="writer"
```

**Result**: Set completo di operazioni Drive mappate.

## Configuration Changes

Scelta architetturale: gogcli come layer di integrazione Google Workspace, con credenziali OAuth 2.0 dedicate.

## Key Discoveries

- Su server headless Ubuntu, il flusso OAuth richiede obbligatoriamente un tunnel SSH — non c'è alternativa senza browser
- google-workspace-mcp è più rapido da configurare ma meno controllabile; gogcli è migliore per produzione
- Usare SEMPRE un account Google dedicato (mai personale) — se il server viene compromesso, l'attaccante ha accesso a tutti i servizi Google collegati
- A inizio 2026 una campagna chiamata "ClawHavoc" ha piazzato skill malevole con nomi quasi identici a quelli legittimi su ClawHub — preferire le 53 skill bundle ufficiali di OpenClaw

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| OAuth flow non completabile su headless | Nessun browser disponibile | Tunnel SSH: `ssh -L 8080:localhost:8080 utente@IP` |
| Rischio sicurezza account Google | Account personale esposto | Usare account Google dedicato e isolato |
| Skill malevole su ClawHub | Campagna ClawHavoc | Usare solo le 53 skill bundle ufficiali |

## Final State

Guida completa preparata in formato .md per il bot. Pronta per essere eseguita step-by-step sull'Agimagic.

## Open Questions

- Quale livello di permessi OAuth è sufficiente? (read-only vs full access per Drive)
- Come gestire il refresh token su un server always-on?
- Possibilità di automatizzare il rinnovo credenziali senza intervento manuale?

---

*Session ready for Kos elaboration.*
