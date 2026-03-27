---
title: "Setup completo: Google Drive struttura, skills gws linkate, tutti i binding testati e funzionanti"
date: "2026-03-25"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "mcp", "mcp-servers", "setup", "troubleshooting"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24 (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Completare il setup dell'infrastruttura 8-agenti: configurare Google Drive con struttura cartelle, linkare skills gws nei workspace, testare tutti i binding su WhatsApp e Telegram, e documentare lo stato finale.

## Context

- OpenClaw 2026.3.23-2 running, gateway attivo
- 8 agenti registrati con workspace
- Account operativo agenti: bot-account@example.com (credenziali Google Cloud)
- gws installato in ~/googleworkspace-cli/
- Tutti i fix binding precedenti applicati

## Steps Taken

### 1. Creazione struttura cartelle Google Drive

Account: bot-account@example.com (owner)

```
Family/
  ├── Documenti
  ├── Foto-documenti
  └── Archivio

WIP/
  ├── Progetti
  ├── Ricerche
  ├── Snippets
  └── Archivio

Work/
  ├── MasterControl
  ├── CSO
  ├── Frontend
  ├── Backend
  ├── OrchArch
  └── Shared
```

**Permessi condivisione**:
- Family/ → owner-primary@example.com + family-member1@example.com (writer)
- WIP/ → owner-primary@example.com + family-member2@example.com (writer)
- Work/ → owner-primary@example.com (writer)

**Result**: Struttura completa con folder ID reali salvati.

### 2. Linkaggio skills gws nei workspace

Skills selettive per agente — non tutti hanno accesso a tutto:

**Kai (family)**:
```
gws-calendar, gws-docs, gws-drive, gws-drive-upload, gws-gmail, gws-gmail-send
```

**Mr Wolf (wip)**:
```
gws-docs, gws-docs-write, gws-drive, gws-drive-upload, gws-sheets, gws-tasks
```

**Master Control (orchestrator)**:
```
gws-docs, gws-drive, gws-shared, gws-workflow
```

**Result**: Skills linkate via symlink nei workspace.

### 3. Fix binding topic per Work Framework

**Problema precedente**: threadId nel peer non valido nello schema.

**Sintassi corretta trovata nei docs**: usare `"groupId:topic:N"` come `id` dentro `peer`.

```json
{
  "type": "route",
  "agentId": "cso",
  "match": {
    "channel": "telegram",
    "accountId": "work",
    "peer": { "kind": "group", "id": "TG_GROUP_ID_WORK:topic:3" }
  }
}
```

**Mappa topic completa**:

| Topic | threadId | Formato peer.id |
|-------|----------|-----------------|
| General/Master Control | None | `TG_GROUP_ID_WORK` |
| CSO | 3 | `TG_GROUP_ID_WORK:topic:3` |
| Frontend | 4 | `TG_GROUP_ID_WORK:topic:4` |
| Backend | 5 | `TG_GROUP_ID_WORK:topic:5` |
| Orch. Architect | 6 | `TG_GROUP_ID_WORK:topic:6` |

### 4. Test completo tutti i canali

| Canale | Agente | Stato | Note |
|--------|--------|-------|------|
| WhatsApp gruppo Family | Kai | ✅ | Risponde correttamente |
| Telegram DM @REDACTED_BOT_1 | Kos (CoS) | ✅ | |
| Telegram @REDACTED_BOT_3 | Mr Wolf | ✅ | Gruppo con Rakki + Mauri |
| Telegram Job-desk General | Master Control | ✅ | |
| Telegram Job-desk CSO | CSO | ✅ | |
| Telegram Job-desk Frontend | Frontend | ✅ | |
| Telegram Job-desk Backend | Backend | ✅ | |
| Telegram Job-desk OrchArch | OrchArch | ✅ | |

**Tutti gli 8 agenti funzionanti su tutti i canali.**

### 5. Tentativo broadcast "All Hands" (abbandonato)

Tentativo di creare un gruppo broadcast con tutti gli agenti. Il broadcast Telegram multi-account non è documentato e non funziona senza binding esplicito. Gruppo eliminato.

## Configuration Changes

- Struttura Google Drive creata con 3 cartelle root + sottocartelle
- Permessi condivisione impostati per famiglia e lavoro
- Skills gws linkate selettivamente nei workspace
- Binding topic funzionanti con sintassi `groupId:topic:N`
- MEMORY.md aggiornati per Kai e Mr Wolf con sezione Drive

## Key Discoveries

- **Sintassi topic nei binding**: `"TG_GROUP_ID_WORK:topic:3"` — il formato `groupId:topic:threadId` è la chiave per il routing per topic
- **Skills selettive per workspace**: Kai non ha bisogno di gws-sheets, Mr Wolf non ha bisogno di gws-gmail — principio del minimo privilegio applicato alle skills
- **gws-shared è obbligatoria**: base condivisa richiesta da tutte le altre skills gws
- **Folder ID Google Drive**: ogni cartella ha un ID univoco (es. `DRIVE_ID_FAMILY`) — salvarlo nel MEMORY.md dell'agente per riferimento rapido
- **Broadcast Telegram multi-account non supportato**: non esiste un modo nativo per mandare un messaggio a tutti gli agenti contemporaneamente
- **Tutti i modelli su gpt-5.4-mini di default**: economico e con tool completi, Claude Max come fallback a $0 aggiuntivo
- **3 bot Telegram = 3 account separati**: ogni bot è un account Telegram indipendente con il proprio token

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Binding topic "Invalid input" | threadId nel peer non supportato come campo separato | Usare `groupId:topic:N` come peer.id |
| Broadcast multi-account fallito | Non supportato nativamente da Telegram/OpenClaw | Usare Master Control come orchestratore |

## Final State

**Infrastruttura completa e funzionante:**
- 8 agenti su 3 layer con binding testati
- Google Drive strutturato con permessi per famiglia e lavoro
- Skills gws linkate selettivamente
- 3 bot Telegram + 1 numero WhatsApp operativi
- Gateway systemd attivo su loopback

**Task rimanenti**:
1. Aggiornare SOUL/AGENTS di Kos (ancora riferimenti vecchi a Shikamaru)
2. MEMORY.md Master Control con sezione Drive
3. Testing completo secondo checklist
4. Backup git workspace
5. (Opzionale) Docker + Tailscale hardening
6. (Opzionale) Ollama per fallback locale

## Open Questions

- Come implementare un "All Hands" broadcast a tutti gli agenti?
- Aggiornamento OpenClaw con supporto nativo broadcast/fan-out?
- Performance 8 agenti su gpt-5.4-mini concorrenti — rate limiting?
