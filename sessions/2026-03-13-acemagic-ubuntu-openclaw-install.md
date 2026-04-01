---
title: "Installazione Ubuntu su AceMagic e setup iniziale OpenClaw"
date: "2026-03-13"
author: "kos-domus"
status: "processed"
tags: ["setup", "configuration", "cli"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (fresh install on AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-sonnet-4-6"
---

## Objective

Installare Ubuntu su un mini PC AceMagic dedicato esclusivamente a OpenClaw, poi installare e configurare OpenClaw con Anthropic come provider AI.

## Context

- Mini PC AceMagic con 16GB RAM, preinstallato con Windows
- Ubuntu precaricato su chiavetta USB
- Il PC sarà dedicato esclusivamente a OpenClaw (headless server)
- Connessione internet via cavo ethernet (WiFi non supportata — chip MediaTek MT7902 manca driver Linux fino a kernel 7.1)

## Steps Taken

### 1. Boot da USB e installazione Ubuntu

Il PC avviava Windows dall'SSD interno. Per fare boot dalla USB:
- Spegnere il PC completamente
- Inserire la chiavetta USB
- All'accensione premere ripetutamente **F7** (su AceMagic) per aprire il Boot Menu
- Selezionare la chiavetta USB

**Nota**: Windows Autopilot può interferire mostrando schermate di provisioning. In quel caso: `Shift + F10` per aprire prompt comandi → `shutdown /s /t 0` oppure `wpeutil shutdown`.

**Result**: Raggiunta la schermata di installazione Ubuntu.

### 2. Scelta tipo di installazione

Opzioni presentate:
- "Install Ubuntu alongside Windows" (dual boot)
- "Erase disk and install Ubuntu" (solo Ubuntu)

Scelta: **Erase disk and install Ubuntu** — il PC è dedicato a OpenClaw, Windows non serve.

**Importante**: Verificare che la destinazione sia l'SSD interno (`sda` o `nvme0n1`), NON la chiavetta USB da cui si è avviato.

**Result**: Ubuntu installato con successo sull'SSD interno.

### 3. Connessione a internet

Il chip WiFi MediaTek MT7902 non ha driver Linux disponibili (arriveranno con kernel 7.1). Alternative:
- **Cavo ethernet** (scelta adottata) — funziona immediatamente
- Dongle WiFi USB (TP-Link TL-WN725N o Archer T3U)
- Tethering USB con smartphone

```bash
# Verifica connessione
ping -c 4 google.com
```

**Result**: Connesso via ethernet.

### 4. Installazione OpenClaw

```bash
# curl non preinstallato su Ubuntu minimal
sudo apt install curl

# Installazione OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash
```

Lo script installa automaticamente Node.js se mancante e avvia l'onboarding wizard.

**Result**: OpenClaw installato con successo.

### 5. Configurazione provider AI (primo tentativo — API key)

Durante l'onboarding, configurato Anthropic come provider con API key (`sk-ant-api03-...`).

**Result**: Funzionante ma con rate limits bassi — vedi sessione separata sull'autenticazione.

## Configuration Changes

- Ubuntu installato su SSD interno AceMagic (Windows rimosso)
- OpenClaw installato via script ufficiale
- Connessione internet via ethernet

## Key Discoveries

- **AceMagic Boot Menu**: tasto F7 (può variare per modello: anche F11, F12, Esc)
- **Windows Autopilot**: può bloccare l'avvio — usare `wpeutil shutdown` per spegnere dall'ambiente di installazione Windows
- **MediaTek MT7902**: nessun driver Linux fino a kernel 7.1 — usare ethernet o dongle USB
- **curl non preinstallato**: su Ubuntu minimal serve `sudo apt install curl` prima dell'installazione
- **Nota sicurezza OpenClaw**: software sperimentale, mai installare come root, usare dispositivo dedicato

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Windows Autopilot si avvia al posto di Ubuntu | Boot order sbagliato | F7 al boot per selezionare USB |
| `command curl not found` | curl non preinstallato | `sudo apt install curl` |
| Nessun WiFi rilevato (`nmcli device status` mostra solo ethernet) | Driver MT7902 mancante | Usare ethernet o dongle USB |
| Tastiera AceMagic problematica (tasti mancanti) | Hardware issue | Usare `onboard` o `gnome-on-screen-keyboard` da terminale |

## Final State

Ubuntu funzionante su AceMagic con OpenClaw installato e connesso via ethernet.

## Open Questions

- Quando arriverà il supporto kernel per MT7902 su Ubuntu?
- Performance di OpenClaw su 16GB RAM con operazioni intensive?
