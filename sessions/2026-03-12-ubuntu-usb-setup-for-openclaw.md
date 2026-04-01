---
title: "Guida completa: installazione Ubuntu su chiavetta USB per mini PC dedicato a OpenClaw"
date: "2026-03-12"
author: "kos-domus"
status: "processed"
tags: ["setup", "configuration", "cli"]
openclaw_version: "latest"
environment:
  os: "Ubuntu 24.04 LTS (fresh install)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Creare una chiavetta USB avviabile con Ubuntu, installarlo su un mini PC dedicato, e configurare il sistema per OpenClaw.

## Context

- Mini PC dedicato esclusivamente a OpenClaw
- Ubuntu scelto come OS
- Installazione da chiavetta USB

## Steps Taken

### 1. Creazione chiavetta USB avviabile

1. Scaricare Ubuntu 24.04 LTS ISO (~5-6 GB) da ubuntu.com/download/desktop
2. Scaricare balenaEtcher da etcher.balena.io
3. Inserire chiavetta USB da almeno 8 GB (contenuto cancellato)
4. In balenaEtcher: Flash from file → selezionare ISO → selezionare USB → Flash!

**Result**: Chiavetta avviabile creata in ~10 minuti.

### 2. Installazione Ubuntu sul mini PC

1. Inserire USB nel mini PC e accendere
2. Premere F2/F10/F12/Del/Esc al boot per accedere al Boot Menu
3. Selezionare USB come dispositivo di boot
4. Scegliere "Install Ubuntu"
5. Opzioni consigliate:
   - "Normal installation" (non minimal)
   - Spuntare "Install third-party software" per driver e codec
   - "Erase disk and install Ubuntu" (PC dedicato a OpenClaw)
6. Impostare nome utente, password, fuso orario

**Result**: Ubuntu installato in ~15 minuti.

### 3. Preparazione sistema

```bash
# Aggiornare tutto
sudo apt update && sudo apt upgrade -y

# Installare dipendenze base
sudo apt install -y curl git build-essential
```

### 4. Installazione Node.js 22+ (requisito OpenClaw)

```bash
# Installare NVM (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Ricaricare il profilo shell
source ~/.bashrc

# Installare Node.js 22 LTS
nvm install 22

# Verificare
node --version   # deve mostrare v22.x.x
npm --version
```

### 5. Installazione OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

L'onboarding wizard chiede:
- **API key**: Anthropic (Claude), OpenAI, o modello locale con Ollama
- **Canale di chat**: WhatsApp, Telegram, Discord, Signal...
- **Skills**: aggiungibili anche dopo
- **Daemon/Gateway**: servizio systemd per funzionamento 24/7

### 6. Verifica

```bash
openclaw status          # stato del sistema
openclaw dashboard       # dashboard web
openclaw doctor          # salute del sistema
openclaw logs --follow   # log in tempo reale
```

**Result**: OpenClaw operativo.

## Configuration Changes

- Ubuntu 24.04 LTS installato
- Node.js 22+ via NVM
- OpenClaw installato con gateway systemd

## Key Discoveries

- **Node.js 22+ è requisito tassativo**: non un suggerimento, serve per il gateway
- **NVM è il modo migliore**: permette di gestire più versioni di Node senza conflitti
- **balenaEtcher** è cross-platform (Mac/Win/Linux) per creare USB avviabili
- **"Install third-party software"** va spuntato: installa driver necessari
- **Requisiti hardware minimi**: 4GB RAM (1GB solo per gateway), ma 8-16GB consigliato
- **Per API cloud (no modelli locali)**: anche mini PC entry-level con 8GB basta
- **Per modelli locali con Ollama**: servono 16-32GB RAM
- **Sicurezza**: partire con permessi minimi, evitare skill di terze parti non verificate, non dare accesso a email/wallet con fondi reali finché non esperti

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Boot Menu non appare | Tasto sbagliato per il modello | Provare F2, F10, F12, Del, Esc |
| `curl: command not found` | Non preinstallato su Ubuntu minimal | `sudo apt install curl` |

## Final State

Mini PC con Ubuntu 24.04 LTS e OpenClaw installato, pronto per configurazione canali e skills.

## Open Questions

- Performance a lungo termine del gateway systemd su mini PC entry-level?
- Backup automatico della configurazione OpenClaw?
