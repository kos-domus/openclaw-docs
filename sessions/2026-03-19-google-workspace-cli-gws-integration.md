---
title: "Integrazione Google Workspace CLI (gws) con OpenClaw per Gmail, Drive, Calendar, Docs, Sheets"
date: "2026-03-19"
author: "kos-domus"
status: "ready"
tags: ["mcp", "mcp-servers", "configuration", "setup", "automation"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Integrare Google Workspace CLI (gws) con OpenClaw per dare al bot accesso operativo completo a Gmail, Calendar, Drive, Docs e Sheets tramite skills dedicate.

## Context

- Account Google dedicato del bot già creato con OAuth configurato
- OpenClaw già operativo su Ubuntu
- Due strumenti disponibili: **gws** (Google, nuovo marzo 2026) e **gogcli** (Peter Steinberger, creatore di OpenClaw)
- gws scelto per integrazione nativa con agenti AI

## Steps Taken

### 1. Disambiguazione gws vs gogcli

| | gws | gogcli |
|---|---|---|
| Autore | Google | Peter Steinberger (creatore OpenClaw) |
| Linguaggio | Node.js | Go |
| Discovery | Dinamico (legge API a runtime) | Statico |
| Orientamento | Agenti AI | CLI scripting |
| Integrazione OpenClaw | Nativa (skills repo) | Via ClawHub |

**Result**: Scelto gws per integrazione nativa e compatibilità futura.

### 2. Installazione gws

```bash
# Prerequisito: Node.js 18+
node --version

# Se necessario aggiornare
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Installazione gws
npm install -g @googleworkspace/cli
gws --version
```

**Result**: gws installato.

### 3. Setup Google Cloud Console (se non già fatto)

1. Creare progetto su console.cloud.google.com
2. Abilitare API: Gmail, Calendar, Drive, Sheets
3. OAuth consent screen → External → aggiungere mail bot come test user
4. Credentials → OAuth Client ID → Desktop app → scaricare JSON

```bash
mkdir -p ~/.config/gws
mv ~/Downloads/client_secret_*.json ~/.config/gws/client_secret.json
```

### 4. Autenticazione

```bash
# IMPORTANTE: specificare sempre i servizi con -s
# Le app in testing mode hanno limite ~25 scope, il login generico ne richiede 85+
gws auth login -s drive,gmail,calendar,sheets
```

Per server headless (senza browser):
```bash
# Su PC locale con browser
gws auth export --unmasked > credentials.json

# Copiare sul server
scp credentials.json utente@server:~/.config/gws/credentials.json
```

**Nota**: Quando appare "Google hasn't verified this app" → Avanzate → Vai a (non sicuro). Normale per uso personale.

**Result**: Autenticazione completata.

### 5. Test connessione

```bash
gws gmail +triage              # email non lette
gws calendar +agenda           # eventi di oggi
gws drive files list --params '{"pageSize": 5}'  # file Drive
```

### 6. Installazione skills per OpenClaw

```bash
# Clonare il repo skills
git clone https://github.com/googleworkspace/cli.git ~/gws-cli

# Creare cartella skills
mkdir -p ~/.openclaw/skills

# OBBLIGATORIO: base condivisa
ln -s ~/gws-cli/skills/gws-shared ~/.openclaw/skills/gws-shared

# Gmail
ln -s ~/gws-cli/skills/gws-gmail ~/.openclaw/skills/gws-gmail
ln -s ~/gws-cli/skills/gws-gmail-send ~/.openclaw/skills/gws-gmail-send
ln -s ~/gws-cli/skills/gws-gmail-triage ~/.openclaw/skills/gws-gmail-triage
ln -s ~/gws-cli/skills/gws-gmail-watch ~/.openclaw/skills/gws-gmail-watch

# Calendar
ln -s ~/gws-cli/skills/gws-calendar ~/.openclaw/skills/gws-calendar
ln -s ~/gws-cli/skills/gws-calendar-agenda ~/.openclaw/skills/gws-calendar-agenda
ln -s ~/gws-cli/skills/gws-calendar-insert ~/.openclaw/skills/gws-calendar-insert

# Drive
ln -s ~/gws-cli/skills/gws-drive ~/.openclaw/skills/gws-drive
ln -s ~/gws-cli/skills/gws-drive-upload ~/.openclaw/skills/gws-drive-upload

# Docs
ln -s ~/gws-cli/skills/gws-docs ~/.openclaw/skills/gws-docs
ln -s ~/gws-cli/skills/gws-docs-write ~/.openclaw/skills/gws-docs-write

# Sheets
ln -s ~/gws-cli/skills/gws-sheets ~/.openclaw/skills/gws-sheets
ln -s ~/gws-cli/skills/gws-sheets-append ~/.openclaw/skills/gws-sheets-append
ln -s ~/gws-cli/skills/gws-sheets-read ~/.openclaw/skills/gws-sheets-read

# Workflow (opzionale ma consigliato)
ln -s ~/gws-cli/skills/gws-workflow ~/.openclaw/skills/gws-workflow
ln -s ~/gws-cli/skills/gws-workflow-standup-report ~/.openclaw/skills/gws-workflow-standup-report
ln -s ~/gws-cli/skills/gws-workflow-meeting-prep ~/.openclaw/skills/gws-workflow-meeting-prep
ln -s ~/gws-cli/skills/gws-workflow-weekly-digest ~/.openclaw/skills/gws-workflow-weekly-digest
ln -s ~/gws-cli/skills/gws-workflow-email-to-task ~/.openclaw/skills/gws-workflow-email-to-task
```

### 7. Configurazione openclaw.json

```json5
{
  "skills": {
    "load": {
      "watch": true  // rileva nuove skills automaticamente
    }
  }
}
```

### 8. Riavvio e test

```bash
openclaw gateway restart
```

Test via WhatsApp: "Quante email non lette ho?", "Cosa ho in agenda oggi?", "Carica un file su Drive"

**Result**: Integrazione completa.

## Configuration Changes

- gws installato globalmente via npm
- Skills collegate via symlink a ~/.openclaw/skills/
- `skills.load.watch: true` in openclaw.json per aggiornamento automatico

## Key Discoveries

- **gws usa Discovery Service dinamico**: non ha lista comandi statica, legge le API Google a runtime → si aggiorna da solo quando Google aggiunge endpoint
- **`gws auth login` senza `-s` fallisce in testing mode**: richiede 85+ scope, limite è ~25. Sempre specificare i servizi
- **"Google hasn't verified this app" è normale**: per uso personale/testing, cliccare Avanzate → Vai a (non sicuro)
- **Symlink > copia**: i symlink si aggiornano automaticamente con `cd ~/gws-cli && git pull`
- **`gws-shared` è obbligatoria**: base condivisa richiesta da tutte le altre skills
- **Skills condivise tra agenti**: `~/.openclaw/skills/` è globale per tutti gli agenti sulla stessa macchina
- **Rischio ban Gmail**: Google può sospendere account per comportamento da bot. Account dedicato mitiga il rischio
- **gws è "developer sample"**: Google non lo supporta ufficialmente, ma è lo standard emergente per agenti AI

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| OAuth login fallisce con troppe scope | Testing mode ha limite ~25 scope | Usare `gws auth login -s drive,gmail,calendar,sheets` |
| "Google hasn't verified this app" | App in testing mode | Avanzate → Vai a (non sicuro) |
| Skills non rilevate da OpenClaw | Cartella o symlink mancanti | Verificare `ls -la ~/.openclaw/skills/`, riavviare gateway |

## Final State

gws integrato con OpenClaw: bot può operare su Gmail, Calendar, Drive, Docs e Sheets tramite conversazione WhatsApp.

## Open Questions

- Performance di gws su 16GB RAM con richieste concorrenti?
- Come gestire il rate limiting delle API Google Workspace?
- Possibilità di aggiungere Google Contacts e Tasks in futuro?
