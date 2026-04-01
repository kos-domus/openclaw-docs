---
title: "Il concetto di Workspace in OpenClaw e agenti indipendenti con isolamento per-canale"
date: "2026-03-20"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "configuration", "concepts", "skills", "mcp"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Comprendere il concetto di Workspace in OpenClaw, creare agenti indipendenti sganciati dal framework gerarchico, e configurare l'isolamento delle cartelle Google Drive per agente.

## Context

- Framework gerarchico a 5 agenti già progettato (master-orchestrator, cso, frontend, backend, orchestration-architect)
- Necessità aggiuntiva: agenti indipendenti per contesti diversi (personale, ricerca, etc.)
- Google Workspace CLI (gws) già configurato e funzionante
- Obiettivo: ogni agente accede solo alla propria cartella su Drive

## Steps Taken

### 1. Comprensione del Workspace

Il workspace è la "casa" dell'agente — l'unica directory di lavoro per file tool e contesto.

**Posizione default**: `~/.openclaw/workspace`, configurabile via `agents.defaults.workspace`.

**File bootstrap iniettati nel prompt ogni sessione**:
- `AGENTS.md` — ruolo e regole dell'agente
- `SOUL.md` — personalità e stile
- `TOOLS.md` — strumenti disponibili
- `USER.md` — contesto utente
- `skills/<skill>/SKILL.md` — skill workspace-specifiche

**Importante**: il workspace non è un sandbox rigido. I path relativi si risolvono rispetto al workspace, ma i path assoluti possono raggiungere altre parti del sistema.

**Best practice dalla community**: tenerlo minimale, metterlo in un repo git privato per backup.

**Result**: Concetto di workspace compreso.

### 2. Creazione agenti indipendenti

Agenti indipendenti = completamente separati dal framework gerarchico, con workspace, sessioni e canali propri.

```bash
# Creare agente con workspace dedicato
openclaw agents add nome-agente --workspace ~/.openclaw/workspace-nome-agente

# Verificare
openclaw agents list --bindings

# Collegare a un canale specifico
openclaw agents bind --agent nome-agente --bind telegram:nome-account
```

Configurazione in `openclaw.json`:

```json
{
  "agents": {
    "list": [
      {
        "id": "master-orchestrator",
        "workspace": "~/.openclaw/workspace/agents/master-orchestrator",
        "model": "anthropic/claude-opus-4-6"
      },
      {
        "id": "personal-assistant",
        "workspace": "~/.openclaw/workspace-personal",
        "model": "anthropic/claude-sonnet-4-6"
      }
    ]
  },
  "bindings": [
    { "agentId": "master-orchestrator", "match": { "channel": "telegram", "accountId": "work" } },
    { "agentId": "personal-assistant",  "match": { "channel": "whatsapp", "accountId": "personal" } }
  ]
}
```

**Tool policy per agente**:
```json
{
  "id": "research-bot",
  "tools": {
    "allowlist": ["browser", "web_search", "read", "write"],
    "denylist": ["exec", "process"]
  },
  "sandbox": { "mode": "always" }
}
```

**Result**: Architettura agenti indipendenti definita.

### 3. Isolamento cartelle Google Drive per agente

**Problema**: Google OAuth non supporta scope per cartella. L'accesso a Drive è binario (tutto o niente).

**Tre approcci valutati**:

| Approccio | Email separate? | OAuth separati? | Isolamento reale? | Complessità |
|-----------|----------------|-----------------|-------------------|-------------|
| AGENTS.md rules | No | No (copia auth-profiles) | Comportamentale | Bassa |
| Service Account | No | Sì (JSON) | Tecnico | Media |
| gws skill per workspace | No | No | Comportamentale + modulare | Bassa |

**Scelta**: Approccio 3 (gws skill per workspace) — già avendo gws configurato, l'isolamento è solo questione di quali skill installare dove.

```bash
# Sub-agenti: solo Drive
ln -s ~/gws-cli/skills/gws-drive ~/.openclaw/workspace-frontend-specialist/skills/gws-drive

# Master Orchestrator: tutte le skill
ln -s ~/gws-cli/skills/gws-* ~/.openclaw/workspace-master-orchestrator/skills/
```

Poi in ogni AGENTS.md, regola esplicita di scope:
```markdown
## Google Drive
You MUST operate exclusively within Drive:/agents/frontend-specialist/
Never read, write, or list files outside this folder.
```

Per il Master Orchestrator:
```markdown
## Google Drive
You have full read/write access to all agent folders under Drive:/agents/
```

**Condivisione credenziali tra agenti** (necessaria una tantum):
```bash
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/frontend-specialist/agent/auth-profiles.json
```

**Result**: Isolamento Drive configurato via skill + regole AGENTS.md.

## Configuration Changes

- Agenti indipendenti definiti in `openclaw.json` con workspace separati
- Bindings canale → agente configurati
- Tool policy per-agente (allowlist/denylist)
- gws skill selettive per workspace
- Regole Drive scope in AGENTS.md

## Key Discoveries

- **Workspace = cervello persistente**: directory da cui l'agente legge istruzioni, scrive file, costruisce contesto tra sessioni
- **Workspace non è sandbox**: path assoluti possono comunque raggiungere il sistema. Per sandbox vero servono permessi OS
- **Agenti indipendenti sono primitiva di primo livello**: non una variante della gerarchia, ma un concetto parallelo
- **Isolamento Drive è comportamentale con gws**: Google OAuth non supporta scope per cartella, ma regole in AGENTS.md + skill selettive danno isolamento sufficiente per uso personale
- **Le credenziali non sono condivise automaticamente**: bisogna copiare `auth-profiles.json` manualmente nell'agentDir di ogni agente
- **Skill globali vs workspace**: `~/.openclaw/skills/` = globale (tutti gli agenti), `workspace/skills/` = solo quell'agente
- **`openclaw agents add` scaffolda tutto**: crea workspace e directory agente, riducendo rischio di condivisione accidentale
- **Due agenti che condividono directory = rottura auth e session history**

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Agente accede a file fuori dal suo scope Drive | Isolamento solo comportamentale | Aggiungere regole esplicite in AGENTS.md + monitorare |
| Due agenti condividono directory per errore | Creazione manuale senza `openclaw agents add` | Usare sempre il CLI per scaffolding |

## Final State

Architettura definita con coesistenza di framework gerarchico (lavoro) e agenti indipendenti (contesti personali), con isolamento Drive via gws skill + regole AGENTS.md.

## Open Questions

- Quanto è affidabile l'isolamento comportamentale in produzione? Servono guardrail tecnici aggiuntivi?
- Come monitorare se un agente sfora il proprio scope Drive?
- Service Account come upgrade futuro per isolamento tecnico reale?
