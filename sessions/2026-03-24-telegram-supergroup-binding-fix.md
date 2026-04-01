---
title: "Configurazione Telegram supergroup con topics, binding per agente e fix WhatsApp peer routing"
date: "2026-03-24"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "configuration", "troubleshooting", "setup"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Creare un supergroup Telegram "Work Framework" con topics dedicati per ogni agente del framework lavorativo, configurare i binding topic→agente, e correggere il routing WhatsApp di Kai.

## Context

- 8 agenti configurati e registrati
- Tre bot Telegram: @REDACTED_BOT_1 (CoS), @REDACTED_BOT_2 (Work Framework), @REDACTED_BOT_3 (Mr Wolf)
- WhatsApp con gruppo "Family stuff" per Kai
- Binding iniziali incompleti o errati

## Steps Taken

### 1. Fix binding WhatsApp per Kai

**Problema**: Il binding usava il JID del gruppo come `accountId` invece di `peer`. Kos (CoS) intercettava tutti i messaggi perché era il default.

**Errore**:
```json
{
  "agentId": "family",
  "match": {
    "channel": "whatsapp",
    "accountId": "WA_GROUP_JID_FAMILY@g.us"  // SBAGLIATO
  }
}
```

**Fix**:
```json
{
  "agentId": "family",
  "match": {
    "channel": "whatsapp",
    "accountId": "default",
    "peer": { "kind": "group", "id": "WA_GROUP_JID_FAMILY@g.us" }  // CORRETTO
  }
}
```

Applicato via script Python che modifica openclaw.json in-place.

**Result**: Kai risponde solo nel gruppo Family stuff, Kos non intercetta più.

### 2. Creazione supergroup Telegram "Work Framework"

1. Creato gruppo "Job-desk" su Telegram
2. Abilitata modalità Forum/Topics
3. Promosso @REDACTED_BOT_2 ad admin
4. Creati 6 topics: All, Master Control, CSO, Frontend, Backend, Orch. Architect

**Nota BotFather**: Privacy mode deve essere disabilitato per i bot in gruppo.
- BotFather → /setprivacy → seleziona bot → Disable
- Senza questo, i bot non ricevono messaggi di gruppo

### 3. Recupero threadId dei topics

**Problema**: I log di OpenClaw non mostravano i threadId — il gateway consumava gli update con polling prima che fossero leggibili.

**Soluzione**: Fermare il gateway, mandare messaggi nei topics, leggere gli update direttamente dalla Bot API.

```bash
# Fermare il gateway
systemctl --user stop openclaw-gateway.service

# Mandare un messaggio con il nome del topic in ogni topic

# Leggere gli update
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN_WORK}/getUpdates?limit=50" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for u in data.get('result', []):
    msg = u.get('message', {})
    thread = msg.get('message_thread_id')
    text = msg.get('text', '')[:40]
    print(f'threadId={thread} text={text!r}')
"
```

**Mappa risultante**:

| Topic | threadId |
|-------|----------|
| General / Master Control | None (topic principale del supergroup) |
| CSO | 3 |
| Frontend | 4 |
| Backend | 5 |
| Orch. Architect | 6 |

**chatId**: TG_GROUP_ID_WORK

### 4. Configurazione binding topic → agente

**Problema 1**: Il CLI `openclaw agents bind` non supporta la sintassi topic/thread.

**Problema 2**: Binding con `threadId` nel peer causava errori di validazione ("Invalid input").

**Soluzione finale**: OpenClaw non supporta nativamente il routing per threadId nel binding schema. Il workaround è usare un singolo binding per il gruppo intero verso Master Control, che poi delega internamente.

**Binding funzionante**:
```json
{
  "type": "route",
  "agentId": "orchestrator",
  "match": {
    "channel": "telegram",
    "accountId": "work",
    "peer": { "kind": "group", "id": "TG_GROUP_ID_WORK" }
  }
}
```

### 5. Aggiornamento architettura

Cambiamenti rispetto al design originale:

| Elemento | Prima | Dopo |
|----------|-------|------|
| Master Orchestrator | Spawned da Kos via sessions_spawn | Binding Telegram diretto (rinominato Master Control) |
| Kos (CoS) | Shikamaru, WhatsApp DM + Telegram DM | Solo Telegram DM @REDACTED_BOT_1 |
| Kos (wip) | WhatsApp gruppo WIP | Rinominato Mr Wolf, Telegram @REDACTED_BOT_3 |
| Framework lavorativo | Proxy di Kos | Bot separato @REDACTED_BOT_2 |
| WhatsApp | Multi-agent sullo stesso numero | Solo Kai (Family stuff) su WhatsApp |

## Configuration Changes

- Binding WhatsApp Kai corretto con `peer.kind: "group"`
- Supergroup Telegram "Job-desk" creato con 6 topics
- @REDACTED_BOT_2 privacy mode disabilitato
- Binding orchestrator su telegram:work con peer group
- Aggiornato modello agenti: default openai/gpt-5.4-mini, Mr Wolf gpt-5.4

## Key Discoveries

- **accountId ≠ JID del gruppo**: il binding WhatsApp deve usare `peer.kind: "group"` per filtrare per gruppo, non mettere il JID nell'accountId
- **OpenClaw non supporta threadId nei binding**: il routing per topic Telegram non è nativamente supportato nello schema binding
- **Bot Telegram privacy mode**: DEVE essere disabilitato in BotFather per ogni bot che deve ricevere messaggi di gruppo
- **Topic General ha threadId=None**: il primo topic del supergroup non ha thread_id esplicito
- **Il gateway consuma gli update**: per leggere i threadId dalla Bot API, bisogna prima fermare il gateway
- **`openclaw agents bind` è limitato**: non supporta sintassi peer con kind/group/thread — modificare il JSON direttamente
- **WhatsApp non supporta multi-agent sullo stesso numero**: tutti gli agenti non-family spostati su Telegram
- **Due servizi gateway contemporanei**: openclaw-gateway.service e openclaw-node.service possono coesistere — usare solo uno
- **openclaw.json validazione rigorosa**: campi non previsti nello schema (es. threadId nel peer) causano "Invalid input"
- **Script Python per modifiche JSON**: più sicuro del nano manuale — valida prima di scrivere

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Kos risponde al posto di Kai su WhatsApp | accountId usato come JID invece di peer | Aggiungere `peer.kind: "group"` nel binding |
| "not-allowed" nei log per Job-desk | Gruppo non nella allowlist del bot work | Aggiungere gruppo e groupPolicy in config telegram accounts |
| binding "Invalid input" | threadId non supportato nello schema | Usare binding a livello gruppo, senza threadId |
| "Skipped bindings already claimed by another agent" | Binding generico di orchestrator intercetta tutto | Rimuovere binding generico prima di aggiungere specifici |
| Log non mostrano messaggi Telegram recenti | Gateway consuma update con polling | Fermare gateway, leggere update via Bot API, riavviare |

## Final State

- Kai operativo su WhatsApp gruppo Family stuff
- Kos (CoS) operativo su Telegram DM @REDACTED_BOT_1
- Mr Wolf operativo su Telegram @REDACTED_BOT_3 con gruppo WIP
- Master Control (orchestrator) binding su Telegram @REDACTED_BOT_2 supergroup Job-desk
- Sub-agenti (CSO, Frontend, Backend, OrchArch) non hanno binding per topic (non supportato) — gestiti da Master Control

## Open Questions

- Come implementare il routing per topic se OpenClaw non lo supporta nativamente? (Custom middleware? Plugin?)
- Come far comunicare Master Control con i sub-agenti senza binding topic separati?
- Possibilità di aggiornamento OpenClaw con supporto threadId nei binding?
