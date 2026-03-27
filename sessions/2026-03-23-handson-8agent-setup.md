---
title: "Setup hands-on: creazione 8 workspace, file .md e openclaw.json su mini PC Ubuntu"
date: "2026-03-23"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "setup", "cli", "troubleshooting"]
openclaw_version: "2026.3.13"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Eseguire la configurazione completa degli 8 agenti su OpenClaw direttamente da terminale Ubuntu: creare workspace, scrivere tutti i file .md (IDENTITY, SOUL, AGENTS, TOOLS, USER, HEARTBEAT, BOOT, FLEET), registrare gli agenti e configurare openclaw.json.

## Context

- OpenClaw 2026.3.13 running, gateway attivo su LAN 192.168.1.87:18789
- Node 22.22.1, systemd service attivo
- API Keys pronte: Anthropic, OpenAI, Telegram
- WhatsApp linked con due gruppi mappati
- Setup guide da 3480 righe con 13 TASK da completare
- Dati reali già risolti (JID WhatsApp, numeri autorizzati, Telegram user ID)

## Steps Taken

### 1. Impostazione variabili d'ambiente

```bash
export RAKKI_WA="393403614393@s.whatsapp.net"
export FAMILY_JID="120363406285133983@g.us"
export WIP_JID="120363424787771105@g.us"
export TG_ID="tg:877281564"
export DATA_SETUP="2026-03-23"
```

**Dati reali mappati**:
- Rakki WhatsApp: +393403614393
- Katia (ragazza): +393495815883
- Mauri (padre): +393479196887
- Family stuff group JID: 120363406285133983@g.us
- Work in progress group JID: 120363424787771105@g.us
- Rakki Telegram user ID: 877281564 (username: Rakki_Otoko)
- Secondo Telegram ID: 589106050 (da confermare chi è)

**Result**: Variabili impostate, nessun output = OK.

### 2. Creazione 8 workspace e directory

```bash
for ws in cos orchestrator cso frontend-specialist backend-expert orchestration-architect family wip; do
  mkdir -p ~/.openclaw/workspace-$ws/memory
  mkdir -p ~/.openclaw/workspace-$ws/procedures
  mkdir -p ~/.openclaw/workspace-$ws/skills
done
```

**Result**: Tutti e 8 creati con successo.

### 3. Registrazione agenti in OpenClaw

```bash
openclaw agents add <id> --workspace ~/.openclaw/workspace-<id>
```

Per tutti e 8 gli agenti. Il comando crea automaticamente:
- Workspace directory (se non esiste)
- Sessions directory: `~/.openclaw/agents/<id>/sessions`
- Agent directory: `~/.openclaw/agents/<id>/agent`

**Nota**: `family` era già esistente dalla configurazione precedente — nessun errore, il comando skippa.

**Result**: 8 agenti registrati.

### 4. Scrittura file .md per Shikamaru (CoS)

File creati con `cat > file << 'EOF'`:
- IDENTITY.md (6 righe)
- SOUL.md (51 righe) — personalità, stile, principi, dominio, fleet protocol
- AGENTS.md (91 righe) — fleet registry, lifecycle protocol, trust levels, routing, security, memory system
- TOOLS.md (35 righe) — filesystem, runtime, sessions, web, memory
- USER.md — contesto utente con numeri autorizzati e routing per canale
- HEARTBEAT.md — checklist fleet health, config drift, security, memory maintenance
- BOOT.md (12 righe) — sequenza startup gateway
- memory/FLEET.md — registro stato agenti con canali e changelog
- skills/agent-configurator/SKILL.md — workflow per aggiungere/modificare/rimuovere agenti

**Result**: Workspace Shikamaru completo.

### 5. Problemi incontrati con heredoc e paste

**Problema 1**: Backtick dentro heredoc confonde il parser bash.
- **Soluzione**: Usare delimitatore diverso (es. `'SKILLEOF'` invece di `'EOF'`)

**Problema 2**: Virgolette tipografiche (' ' — " ") copiate da Claude UI non funzionano in bash.
- **Soluzione**: Usare solo ASCII standard (' - ")

**Problema 3**: Paste multi-linea si tronca nel terminale su file lunghi.
- **Soluzione**: Usare `python3 -c "open('path', 'w').write('''contenuto''')"` che non ha limiti di paste

**Problema 4**: File .md sovrascritta dal template default di OpenClaw al primo avvio.
- **Soluzione**: Scrivere i file DOPO `openclaw agents add`, poi verificare con `tail` e `wc -l`

## Configuration Changes

- 8 workspace creati in `~/.openclaw/workspace-*`
- 8 agenti registrati in `openclaw.json`
- File .md completi per Shikamaru (CoS)
- File in lavorazione per gli altri 7 agenti

## Key Discoveries

- **`openclaw agents add` crea file default**: HEARTBEAT.md e USER.md vengono pre-popolati con template default — i file custom vanno scritti DOPO il comando
- **heredoc con backtick**: se il contenuto del file contiene backtick (es. blocchi codice), il heredoc si confonde. Usare delimitatore diverso o python3
- **Virgolette Unicode vs ASCII**: Claude UI genera virgolette tipografiche che bash non riconosce come quote. Sempre verificare
- **Paste lungo si tronca**: il terminale Ubuntu ha un limite di buffer per paste. Per file lunghi, usare python3 o scp
- **Workspace extra dalla vecchia config** (workspace-katia, workspace-work, workspace) non danno problemi — si possono lasciare
- **OpenClaw banner cambia ogni volta**: ogni comando `openclaw agents add` mostra un motto diverso (es. "Your config is valid, your assumptions are not")
- **`wc -l` + `tail -3`**: modo rapido per verificare integrità dei file dopo la scrittura

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| heredoc non si chiude | Backtick nel contenuto | Usare delimitatore diverso: `<< 'SKILLEOF'` |
| File si tronca durante paste | Buffer terminale limitato | Usare `python3 -c "open().write()"` |
| USER.md contiene template default | `openclaw agents add` sovrascrive | Scrivere file custom DOPO il comando |
| Virgolette non riconosciute | Unicode tipografico da Claude UI | Usare solo ASCII: ' e " |

## Final State

8 workspace creati e agenti registrati. File .md di Shikamaru completi. TASK rimanenti da completare per gli altri 7 agenti: Master Orchestrator, CSO, Frontend, Backend, Orchestration Architect, Kai, Kos.

## Open Questions

- Come automatizzare la scrittura dei file .md per evitare problemi di paste?
- Script bash pre-compilato vs esecuzione passo-passo: quale approccio è più sicuro?
- Come gestire il rollback se un file viene scritto in modo incompleto?
