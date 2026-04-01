---
title: "Configurazione WhatsApp per OpenClaw con Baileys e eSIM dedicata"
date: "2026-03-18"
author: "kos-domus"
status: "processed"
tags: ["configuration", "automation", "setup", "security"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Configurare il bot OpenClaw per operare su WhatsApp con un numero eSIM dedicato, permettendogli di inviare messaggi, creare gruppi e interagire con membri autorizzati.

## Context

- Il bot ha una eSIM dedicata (numero proprio)
- Il proprietario usa WhatsApp standard sul proprio numero personale
- Serve separazione: bot su WhatsApp Business, proprietario su WhatsApp normale
- OpenClaw usa Baileys (WhatsApp Web protocol) di default — nessuna API Meta necessaria

## Steps Taken

### 1. Valutazione approcci: Business API vs WhatsApp Business App

**WhatsApp Business API (Meta)**: richiede approvazione Meta, template pre-approvati per messaggi proattivi, più complessa da configurare. Adatta a grandi volumi.

**WhatsApp Business App via Baileys** (scelta adottata): nessuna approvazione necessaria, nessun template, operativa in minuti, il bot può scrivere a chiunque liberamente. Standard de facto per bot personali/piccola scala.

**Result**: Scelta Baileys tramite OpenClaw nativo.

### 2. Configurazione WhatsApp in OpenClaw

```bash
# Avviare l'onboarding
openclaw onboard

# Oppure in modalità headless (QR code ASCII nel terminale)
openclaw channels login --channel whatsapp --qr-terminal
```

Inserire il numero personale del proprietario quando richiesto ("the phone you will message from") — NON il numero del bot.

Scansionare il QR code con WhatsApp sul telefono con la eSIM:
Impostazioni → Dispositivi collegati → Collega dispositivo

**Result**: Bot connesso a WhatsApp.

### 3. Configurazione openclaw.json

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["+39XXXXXXXXXX"],
      "groupPolicy": "allowlist",
      "autoRead": true,
      "typingIndicator": true
    }
  }
}
```

**Parametri chiave**:
- `dmPolicy: "allowlist"` → solo numeri in `allowFrom` possono scrivere in DM
- `groupPolicy: "allowlist"` → controllo su chi può interagire nei gruppi
- `allowFrom` → numero personale del proprietario (formato E.164: +39...)
- `allowGroups` → numeri autorizzati a interagire nei gruppi (aggiungibili in seguito)

**Result**: Configurazione applicata.

### 4. Recupero Group ID per configurazione gruppi

Il Group ID di WhatsApp ha formato: `XXXXXXXXXX-XXXXXXXXXX@g.us`

**Metodi per recuperarlo** (in ordine di praticità):
1. **Log di OpenClaw** (più semplice): mandare un messaggio nel gruppo, poi `tail -f /tmp/openclaw/openclaw.log` e cercare `@g.us`
2. **grep sui log**: `grep -i "g.us" /tmp/openclaw/openclaw.log`
3. **Baileys/whatsapp-web.js**: query programmatica dei gruppi attivi
4. **WhatsApp Web DevTools**: Console → comandi JS (ma instabile tra versioni)
5. **IndexedDB nel browser**: Application → IndexedDB in DevTools

**Nota**: WhatsApp Web è una SPA — l'URL non cambia navigando tra chat, quindi non si può leggere l'ID dall'URL. I DevTools Elements non espongono i Group ID nel DOM.

**Result**: Group ID recuperabile dai log.

## Configuration Changes

- WhatsApp abilitato in OpenClaw via Baileys
- Policy allowlist per DM e gruppi
- QR code scanning per autenticazione sessione

## Key Discoveries

- **OpenClaw usa Baileys nativamente**: nessuna API Meta, nessun costo, nessuna approvazione
- **"Personal number" nell'onboarding = il TUO numero**: non quello del bot — è l'allowlist
- **La eSIM non può fare doppio duty**: usarla sia per 2FA Google che per WhatsApp Business può creare conflitti. Meglio due eSIM separate o cambiare 2FA Google su authenticator app
- **dmPolicy vs groupPolicy**: DM = conversazioni dirette, gruppi = conversazioni multi-party. Due policy separate per controllo granulare
- **QR code scade in 1-2 minuti**: tenersi pronti, rilanciare il comando se scade
- **Sessione WhatsApp scade dopo 14 giorni offline**: se il telefono con eSIM resta offline troppo a lungo, serve riscansionare il QR
- **Group ID non visibile nell'app**: WhatsApp non espone il Group ID nell'interfaccia utente. Il percorso "Advanced info" suggerito da alcuni bot non esiste
- **Attenzione alle allucinazioni del bot**: OpenClaw può auto-prescrivere modifiche alla propria configurazione con parametri inventati (es. `visibility: "all"`, `pruning: "auto"`) — verificare sempre nella documentazione ufficiale prima di modificare file di config

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Bot suggerisce parametri di config inesistenti | Allucinazione LLM | Verificare sempre sulla documentazione ufficiale |
| QR code scaduto | Timeout 1-2 minuti | Rilanciare `openclaw channels login` |
| Group ID non trovato nell'URL di WhatsApp Web | SPA, URL non cambia | Usare log di OpenClaw o DevTools Console |

## Final State

Bot operativo su WhatsApp con eSIM dedicata, policy di accesso configurate, metodo per recuperare Group ID identificato.

## Open Questions

- Come automatizzare la riscansione del QR code quando la sessione scade?
- Possibilità di usare due eSIM: una per 2FA, una per WhatsApp?
- Come monitorare lo stato della sessione WhatsApp del bot?
