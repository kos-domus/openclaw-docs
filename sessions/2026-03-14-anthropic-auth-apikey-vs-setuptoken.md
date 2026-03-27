---
title: "Autenticazione Anthropic in OpenClaw: API Key vs Setup Token (abbonamento Max)"
date: "2026-03-14"
author: "kos-domus"
status: "ready"
tags: ["configuration", "api", "setup", "troubleshooting"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-sonnet-4-6"
---

## Objective

Risolvere i rate limit raggiunti con OpenClaw configurato con API key Anthropic, passando all'autenticazione via setup-token collegato all'abbonamento Max (90€/mese).

## Context

- OpenClaw configurato inizialmente con API key (`sk-ant-api03-...`)
- Utente ha anche un abbonamento Anthropic Max (90€/mese)
- Rate limits dell'API raggiunti nonostante crediti disponibili
- Il bot non è più raggiungibile via Telegram perché ogni chiamata fallisce con rate limit

## Steps Taken

### 1. Diagnosi del problema rate limit

Errore ricevuto: `API rate limit reached. Please try again later.`
Modello configurato: Sonnet 4.6

Il problema: i **rate limits dell'API** (pagamento a consumo) sono **separati** dall'abbonamento Max. Avere crediti API non significa avere rate limits alti — questi dipendono dal tier di spesa.

**Result**: Identificata la causa — serviva passare all'autenticazione via abbonamento Max.

### 2. Generazione del setup-token

Il setup-token (`sk-ant-oat01-...`) permette di usare l'abbonamento Max invece delle API a consumo.

```bash
# Installare Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Generare il setup-token
claude setup-token
```

Questo apre il browser per il login con l'account Anthropic. Una volta autenticati, genera un token `sk-ant-oat01-...`.

**Importante**: Il token può essere generato su **qualsiasi macchina** (non necessariamente quella dove gira OpenClaw) e poi trasferito.

**Result**: Token generato con successo.

### 3. Riconfigurazione OpenClaw con setup-token

```bash
# Riavviare l'onboarding
openclaw onboard

# Selezionare "Anthropic token (setup-token)"
# NON "Anthropic API key"
# Incollare il token sk-ant-oat01-...
```

Metodo alternativo se non si ha accesso al terminale:
```bash
# Modifica diretta del file di configurazione
nano ~/.openclaw/openclaw.json
```

**Result**: OpenClaw riconfigurato con abbonamento Max — rate limits molto più generosi.

## Configuration Changes

- Autenticazione cambiata da API key (`sk-ant-api03-...`) a setup-token (`sk-ant-oat01-...`)
- Provider: Anthropic via abbonamento Max

## Key Discoveries

- **I rate limits API e abbonamento sono completamente separati**: anche con crediti API, si possono raggiungere i limiti. L'abbonamento Max ha limiti molto più alti
- **Formati token diversi**:
  - `sk-ant-api03-...` = API key (pagamento a consumo, da console.anthropic.com)
  - `sk-ant-oat01-...` = Setup token (collegato all'abbonamento Pro/Max)
- **Il setup-token è portabile**: può essere generato su un Mac e usato su Ubuntu senza problemi
- **Se il bot è irraggiungibile per rate limit**: si può modificare direttamente `~/.openclaw/openclaw.json` o usare la dashboard web su `http://localhost:18789`
- **Per interrompere l'onboarding**: `Ctrl+C` oppure chiudere e riaprire il terminale — l'onboarding riparte da capo con `openclaw onboard`

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `API rate limit reached` | Rate limits API bassi (tier basso) | Passare a setup-token con abbonamento Max |
| OpenClaw chiede `sk-ant-oat01` ma ho `sk-ant-api03` | Selezionato metodo auth sbagliato | Scegliere "Anthropic API key" invece di "setup-token", o meglio: generare il setup-token |
| Bot irraggiungibile dopo rate limit | Tutte le chiamate falliscono | Modificare `~/.openclaw/openclaw.json` direttamente o usare dashboard `localhost:18789` |

## Final State

OpenClaw funzionante con abbonamento Anthropic Max, rate limits generosi, nessun costo API aggiuntivo.

## Open Questions

- Come gestire il refresh del setup-token quando scade?
- È possibile monitorare l'utilizzo dell'abbonamento Max separatamente per OpenClaw?
