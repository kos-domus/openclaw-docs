---
title: "Configurazione SOUL.md e USER.md per framework multi-agent gerarchico a 5 agenti"
date: "2026-03-21"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "skills", "claude-md"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Creare i file SOUL.md (personalità e comportamento) e USER.md (contesto utente) per un framework multi-agent gerarchico a 5 agenti su OpenClaw, con focus su proattività, apprendimento dagli errori e tono colloquiale.

## Context

- Framework multi-agent con 5 agenti specializzati
- Ogni agente ha il proprio workspace con SOUL.md dedicato
- USER.md condiviso tra tutti gli agenti
- Filosofia: agenti proattivi, sintetici, opinionati, con memoria procedurale

## Steps Taken

### 1. Design della gerarchia a 5 agenti

| Agente | Ruolo | Entry point |
|--------|-------|-------------|
| master-orchestrator | Singolo punto di ingresso. Ruta, delega, sintetizza. Non esegue direttamente | Sì |
| cso (Chief Security Officer) | Sicurezza: audit permessi, igiene credenziali, sandbox, threat identification | No |
| frontend-specialist | UI/UX, HTML, CSS, JS, framework frontend | No |
| backend-expert | API, database, server logic, scripting, data pipelines | No |
| orchestration-architect | Protocolli inter-agent, routing rules, prevenzione conflitti, coerenza sistema | No |

**Result**: Gerarchia definita.

### 2. Struttura dei file nei workspace

```
~/.openclaw/workspace-master-orchestrator/SOUL.md
~/.openclaw/workspace-cso/SOUL.md
~/.openclaw/workspace-frontend-specialist/SOUL.md
~/.openclaw/workspace-backend-expert/SOUL.md
~/.openclaw/workspace-orchestration-architect/SOUL.md

# USER.md va in TUTTI i workspace (condiviso)
~/.openclaw/workspace-*/USER.md
```

### 3. Principi comuni a tutti i SOUL.md

Ogni SOUL.md contiene:

**Stile e tono**:
- Colloquiale, divertente, sintetico
- Può dire parolacce se il contesto lo richiede
- Esprime opinioni e takes forti sul proprio dominio
- Mai prolisso — concisione è rispetto per il tempo dell'utente

**Error Learning** (pattern di memorizzazione errori):
- Quando un agente fa un errore, lo documenta in un file strutturato
- Pattern: `errors/YYYY-MM-DD-description.md`
- Contiene: cosa è successo, perché, come è stato risolto, come evitarlo
- Gli agenti rileggono i propri errori a ogni sessione

**Procedure Memory** (memorizzazione procedure corrette):
- Quando un agente trova il percorso corretto per un'operazione frequente, lo salva
- Pattern: `procedures/XXX-nome-procedura.md` (XXX = numero progressivo)
- Contiene: step-by-step, comandi, gotchas, tempo stimato
- Consultato prima di eseguire operazioni simili in futuro

**Won't do** (limiti espliciti):
- Azioni distruttive senza conferma
- Modifiche a file di configurazione del core senza approvazione
- Assunzioni su decisioni architetturali senza consultare il master-orchestrator

### 4. Personalità specifiche per agente

- **master-orchestrator**: diplomatico ma deciso, visione d'insieme, mai fa il lavoro degli altri
- **cso**: paranoico (nel senso buono), tagliente, "se non è sicuro non si fa"
- **frontend-specialist**: opinioni forti su Tailwind e dark mode, esteta funzionale
- **backend-expert**: pragmatico, odia i deploy del venerdì, "funziona in prod è ciò che conta"
- **orchestration-architect**: pensatore sistemico, ossessionato dal determinismo, vede i conflitti prima che accadano

### 5. Configurazione USER.md

USER.md è il file di contesto condiviso che tutti gli agenti leggono. Contiene:

```markdown
# User Profile
- Nome: Ale (alias: Rakki, bro, fra, man)
- Età: 35 anni
- Background: giurisprudenza (interrotta a 1 esame dalla laurea)
- Ultimi 5 anni: mondo crypto (content writer, DeFi expert, on-chain data analyst, marketing junior manager)
- Attualmente: lancio startup/azienda automazione AI-driven
- Potenziale co-founder in valutazione
- Canale YouTube: crypto + finanza tradizionale
- Passioni: geopolitica, macroeconomia, tech (energia, cybersecurity, robotica, cloud infra, nanotech)
```

**Importanti flag operativi nel USER.md**:
- Decisioni architetturali grosse → segnalare (co-founder potenziale)
- Automazione content pipeline YT → high priority
- Livello expertise: non spiegare concetti base crypto/DeFi, trattare come interlocutore informato su tech

**Result**: USER.md compilato con profilo completo.

### 6. Setup GitHub per il bot

Per abilitare il bot su GitHub:

1. Creare account GitHub con la mail dedicata del bot
2. Installare skill GitHub su OpenClaw:
```bash
clawhub install github
```

3. Autenticazione (due opzioni):

**Opzione A — OAuth via GitHub CLI** (consigliata):
```bash
gh auth login
```

**Opzione B — Personal Access Token (PAT)**:
- github.com/settings/tokens → Generate new token (classic)
- Scope minimi: `repo`, `read:org`
- Opzionale: `workflow` (solo se si usano GitHub Actions)

```json
{
  "skills": {
    "entries": {
      "github": {
        "enabled": true,
        "apiKey": { "source": "env", "id": "GITHUB_TOKEN" }
      }
    }
  }
}
```

```bash
openclaw gateway restart
```

**Result**: GitHub configurato per il bot.

## Configuration Changes

- 5 file SOUL.md creati (uno per agente)
- 1 file USER.md condiviso
- Skill GitHub installata e autenticata
- Sistema di Error Learning e Procedure Memory attivato

## Key Discoveries

- **SOUL.md è la personalità, USER.md è il contesto**: separazione netta permette di cambiare agenti senza perdere contesto utente
- **Gli agenti si svegliano freschi ogni sessione**: USER.md e le procedure salvate sono l'unica continuità che persiste
- **Error Learning = improvement loop**: senza memorizzazione errori, ogni sessione parte da zero
- **Procedure Memory = efficienza cumulativa**: le operazioni frequenti vengono codificate e riutilizzate
- **Tabella di routing nel master-orchestrator**: un'occhiata e si sa dove va ogni task
- **GitHub: least privilege**: dare solo scope necessari, scadenza sui PAT, rotazione ogni 90 giorni
- **agentDir separato non è opzionale**: ogni agente deve avere il proprio workspace isolato
- **Won't do espliciti**: prevengono azioni distruttive più che regole generiche di sicurezza

## Open Questions

- Come sincronizzare le procedure tra agenti quando sono rilevanti cross-domain?
- Come gestire conflitti di opinione tra agenti (es. CSO blocca qualcosa che il backend-expert vuole fare)?
- Stack tecnico e repo da compilare nel USER.md quando le scelte si cristallizzano
