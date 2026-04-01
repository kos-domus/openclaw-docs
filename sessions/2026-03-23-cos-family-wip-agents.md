---
title: "Creazione workspace per Chief of Staff (Shikamaru), Family Manager (Kai) e Software Creator (Kos)"
date: "2026-03-23"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "configuration", "skills", "claude-md", "automation"]
openclaw_version: "2026.3.13"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Creare workspace completi per tre nuovi agenti: Shikamaru Nara (Chief of Staff), Kai (Family Manager per gruppo WhatsApp "Family stuff"), e Kos (Software Creator per gruppo WhatsApp "Work in progress").

## Context

- Framework gerarchico lavorativo a 5 agenti già progettato
- Necessità di agenti aggiuntivi per gestione personale/familiare
- Due gruppi WhatsApp: "Family stuff" (fidanzata) e "Work in progress" (padre)
- gws già configurato per accesso Google Workspace
- Ogni agente ha personalità unica e scope definito

## Steps Taken

### 1. Gerarchia complessiva

```
Owner (Ale/Rakki)
  └─ Shikamaru Nara 🦌 [Chief of Staff]
       ├─ Master Orchestrator ⚙️ [framework lavorativo]
       ├─ Kai 🏡 [Family Manager — "Family stuff"]
       └─ Kos 🔧 [Software Creator — "Work in progress"]
```

### 2. Shikamaru Nara — Chief of Staff

**Ruolo**: Secondo in comando dopo il proprietario, configuratore di tutti gli agenti, equiparato al Master Orchestrator ma su ambito diverso (infrastruttura vs lavoro).

**File workspace**:
- `IDENTITY.md` — posizione nella gerarchia, mandate
- `SOUL.md` — personalità (strategico, calmo, pigro-ma-geniale come l'omonimo di Naruto)
- `AGENTS.md` — SOP completo: lifecycle protocol, trust levels, routing
- `USER.md` — profilo owner + authority delegation matrix
- `TOOLS.md` — tool set completo (ha più permessi degli altri agenti)
- `HEARTBEAT.md` — fleet health check ogni 30 minuti
- `BOOT.md` — startup sequence + format primo messaggio
- `memory/FLEET.md` — registro agenti (fonte di verità della flotta)
- `skills/agent-configurator/SKILL.md` — workflow scaffolding nuovi agenti

### 3. Kai — Family Manager

**Ruolo**: Gestione familiare nel gruppo "Family stuff" (proprietario + fidanzata).

**Funzionalità**:
- Gestione contatti mail e telefonici comuni
- Password importanti per la famiglia (vault sicuro)
- Immagini e PDF di documenti
- Tracciamento spese familiari
- Gestione impegni comuni via calendar
- Accesso a Drive, Gmail, Calendar, Sheets, Docs via gws

**Sicurezza**: Niente `exec` né `browser` — deliberato per agente con accesso a dati familiari sensibili.

**File specifici**:
- `memory/VAULT_INDEX.md` — indice posizioni password (niente credenziali in chiaro nel file)

### 4. Kos — Software Creator

**Ruolo**: Supporto generico per sviluppo software nel gruppo "Work in progress" (proprietario + padre).

**Funzionalità**:
- Creazione e gestione progetti software
- Scope emergente (non ancora definito)
- Accesso a Drive, Gmail, Calendar, Sheets, Docs via gws
- `exec` + `browser` abilitati con `exec_ask` attivo

**File specifici**:
- `memory/PROJECTS.md` — tracciamento progetti
- `memory/SCOPE_EVOLUTION.md` — evoluzione dello scope nel tempo

### 5. Deploy checklist

1. **ID gruppi WhatsApp**: da terminale `openclaw channels whatsapp groups`, incollare nel fragment JSON
2. **Placeholder USER.md**: nomi, numeri, email dei membri
3. **OAuth GWS per Kai e Kos**: copiare profilo OAuth dall'agente principale: `cp ~/.openclaw/auth/google-* ~/.openclaw/workspace-family/auth/`
4. **Soglie spesa Kai**: compilare nel USER.md prima di attivare heartbeat mensile

## Configuration Changes

- 3 nuovi workspace creati con tutti i file .md
- Fragment openclaw.json5 con bindings per tutti gli agenti
- Skill agent-configurator per Shikamaru
- FLEET.md come registro centralizzato agenti

## Key Discoveries

- **IDENTITY.md + SOUL.md = chi è l'agente**: separazione tra identità organizzativa e personalità
- **HEARTBEAT.md**: schedule per health check periodici — ogni agente può avere intervalli diversi
- **BOOT.md**: sequenza di startup predefinita che l'agente esegue all'avvio di ogni sessione
- **FLEET.md è la fonte di verità**: registro centralizzato di tutti gli agenti con scope, modello, binding
- **Vault password: solo indice, mai credenziali in chiaro**: VAULT_INDEX.md punta a posizioni, non contiene valori
- **exec_ask è essenziale per agenti con exec**: prompt interattivo previene esecuzioni non autorizzate
- **Regole hard nel config, non solo in AGENTS.md**: AGENTS.md è advisory, openclaw.json è enforcement
- **La dimensione dei file bootstrap impatta i costi token**: ogni sessione carica tutti i file — tenerli sintetici
- **Scope emergente è un pattern valido**: Kos non ha scope definito, lo sviluppa nel tempo tramite SCOPE_EVOLUTION.md

## Open Questions

- Come gestire le soglie di spesa familiare quando cambiano stagionalmente?
- Come sincronizzare FLEET.md tra Shikamaru e il Master Orchestrator?
- Serve un meccanismo di escalation da Kai/Kos verso Shikamaru?
