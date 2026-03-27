---
title: "Creazione account Google dedicato per bot OpenClaw con sicurezza e OAuth"
date: "2026-03-17"
author: "kos-domus"
status: "ready"
tags: ["configuration", "security", "api", "setup"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Creare un account Google dedicato per il bot OpenClaw con accesso completo a Gmail, Calendar, Drive, Places/Maps, configurare OAuth con scope operativi completi e implementare misure di sicurezza (2FA, restrizioni IP, audit log).

## Context

- Il bot OpenClaw gira su AceMagic mini PC con Ubuntu, IP pubblico: 77.32.125.1
- Serve un account Google separato (mai usare account personale per sicurezza)
- Il bot ha una eSIM dedicata per la verifica SMS
- Email di recupero del proprietario: rakkiotokosensei@gmail.com

## Steps Taken

### 1. Creazione account Google dedicato

Istruzioni inviate al bot per creare l'account:
- Username deve contenere "Kos" + "home" o "main" (es. koshome, kosmain)
- Se non disponibili, aggiungere numeri (koshome1, kosmain2...)
- eSIM dedicata disponibile per verifica SMS

**Result**: Account creato dal bot.

### 2. Setup Google Cloud Console

1. Creare un nuovo progetto (es. "KosHome Project")
2. Abilitare le API dalla sezione "API e servizi > Libreria":
   - Google Calendar API
   - Google Drive API
   - Places API
   - Maps JavaScript API
   - Geocoding API
   - Directions API
   - Gmail API

**Result**: Progetto e API configurati.

### 3. Configurazione OAuth con scope completi

Nella schermata OAuth consent (tipo: Esterno), aggiungere i seguenti scope:

| Servizio | Scope | Livello |
|----------|-------|---------|
| Gmail | `https://mail.google.com/` | Accesso completo |
| Calendar | `https://www.googleapis.com/auth/calendar` | Accesso completo |
| Drive | `https://www.googleapis.com/auth/drive` | Accesso completo (NON `.readonly`) |

Creare OAuth 2.0 Client ID e completare il flusso per generare un **refresh token persistente**.

**Nota importante**: Il refresh token è il "lasciapassare permanente" del bot. Non scade (salvo revoca manuale) e va conservato in modo sicuro.

**Result**: OAuth configurato con refresh token.

### 4. Configurazione sicurezza

**2FA**: Attivata con SMS sulla eSIM dedicata.

**Email di recupero**: rakkiotokosensei@gmail.com

**Restrizioni API Key** (per Places/Maps):
- Restrizioni applicazione → Indirizzi IP → `77.32.125.1`
- Restrizioni API → solo Places API, Maps JavaScript API, Geocoding API, Directions API

**Monitoraggio**:
- Audit log attivi in IAM
- Budget alert a 1€ per utilizzo anomalo
- Notifiche sicurezza per accessi da dispositivi/posizioni insolite

**Result**: Sicurezza multi-livello configurata.

## Configuration Changes

- Account Google dedicato creato per il bot
- Progetto Google Cloud con 7 API abilitate
- OAuth Client ID con scope completi (Gmail, Calendar, Drive)
- API Key con restrizione IP per Places/Maps
- 2FA, audit log e budget alert attivi

## Key Discoveries

- **I rate limits API e abbonamento sono separati**: avere crediti non equivale ad avere rate limits alti
- **Places/Maps usa API Key, non OAuth**: meccanismo di autenticazione diverso, più vulnerabile → restrizione IP necessaria
- **OAuth scope non read-only**: per operatività completa del bot, serve `https://www.googleapis.com/auth/drive` (non `.readonly`)
- **Refresh token è permanente**: non scade, revocabile da myaccount.google.com/permissions — è l'"interruttore d'emergenza"
- **IP pubblico ≠ IP locale**: Google Cloud vede l'IP pubblico del router (77.32.125.1), non quello LAN (192.168.1.87)
- **IP geolocalizzazione imprecisa**: IP mostrava Brescia invece di Cisano Bergamasco — normale, non influisce
- **Verifica IP da terminale**: `curl ifconfig.me` restituisce l'IP pubblico corretto
- **App in testing mode**: limite ~25 scope OAuth; massimo 100 utenti di test senza verifica Google

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| IP locale non funziona per restrizioni API | Google Cloud vede IP pubblico | Usare `curl ifconfig.me` per trovare IP pubblico |
| Google blocca verifica per troppi tentativi | Rate limit verifica account | Aspettare 2-4 ore, usare "Try another way" |

## Final State

Account Google dedicato operativo con accesso completo a Gmail, Calendar, Drive e Maps. Sicurezza multi-livello configurata con 2FA, restrizioni IP e audit log.

## Open Questions

- Come gestire il cambio IP pubblico se l'ISP usa IP dinamico? (Considerare IP statico o DuckDNS)
- Google Inactive Account Manager per timeout automatico?
- Monitoraggio attività bot con BCC nascosto su email inviate?
