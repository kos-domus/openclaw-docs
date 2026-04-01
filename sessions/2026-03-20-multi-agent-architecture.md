---
title: "Architettura multi-agent OpenClaw con channel agents isolati per WhatsApp e Telegram"
date: "2026-03-20"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "configuration", "automation", "skills"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Progettare un framework multi-agent su OpenClaw con channel agents completamente sganciati dalla gerarchia di base, ognuno con skills e configurazione propri, per operare in gruppi WhatsApp e Telegram indipendentemente.

## Context

- Bot OpenClaw operativo su Ubuntu con WhatsApp e Google Workspace
- Necessità di agenti separati per canali diversi (WhatsApp, Telegram)
- Ogni agente deve avere: stato isolato, skills dedicati, config file proprio
- Stack: Node.js/TypeScript + Python

## Steps Taken

### 1. Design dell'architettura

Principio fondamentale: **due mondi separati**. I channel agents non importano nulla dal core gerarchico.

```
openclaw/
├── core/                        ← gerarchia base
│   ├── orchestrator.ts
│   ├── agents/
│   │   ├── search.agent.ts
│   │   ├── code.agent.ts
│   │   └── data.agent.ts
│   ├── tools/                   ← MCP tools condivisi
│   └── memory/                  ← Vector DB + session store
│
└── channels/                    ← completamente sganciati
    ├── whatsapp/
    │   ├── index.ts             ← entry point WA (Baileys)
    │   ├── config.wa.yaml       ← system prompt, skills, limiti
    │   ├── skills/              ← tools dedicati per WA
    │   └── agent.wa.ts          ← istanza agent standalone
    │
    └── telegram/
        ├── index.ts             ← entry point TG (grammY)
        ├── config.tg.yaml       ← config separata
        ├── skills/
        └── agent.tg.ts
```

**Result**: Struttura progettata.

### 2. Isolamento dello stato per gruppo

Ogni channel agent usa un `stateKey` con namespace isolato:

```typescript
const agent = new AgentRuntime({
  systemPrompt: config.systemPrompt,
  skills:       waSkills,              // SOLO skills WA
  stateKey:     `wa:${groupId}`,       // namespace per gruppo
  model:        config.model,
  maxTurns:     config.maxTurns,
})
```

Con Redis come state store:
```
wa:123456789@g.us:  → stato gruppo WA "Family"
wa:987654321@g.us:  → stato gruppo WA "Work"
tg:12345678:        → stato chat TG
core:session:xyz    → core non tocca mai wa: o tg:
```

**Result**: Pattern di isolamento definito.

### 3. Config file per canale

```yaml
# channels/whatsapp/config.wa.yaml
model: claude-sonnet-4-6
maxTurns: 10
maxTokens: 2000

systemPrompt: |
  Sei un assistente per il gruppo WhatsApp {groupName}.
  Rispondi solo se menzionato con @bot.
  Usa un linguaggio informale e conciso.

skills:
  - summarize
  - poll

permissions:
  webSearch: false
  codeExec: false
```

Ogni canale ha config YAML separata con system prompt, skills e permessi propri.

**Result**: Template di config creato.

### 4. Deploy separato

```json
{
  "scripts": {
    "start:core":      "ts-node core/orchestrator.ts",
    "start:whatsapp":  "ts-node channels/whatsapp/index.ts",
    "start:telegram":  "ts-node channels/telegram/index.ts"
  }
}
```

Ogni channel agent è un processo indipendente: puoi fermare/aggiornare WhatsApp senza toccare Telegram o il core.

**Result**: Architettura di deploy definita.

## Configuration Changes

- Struttura cartelle con separazione core/channels
- Config YAML per canale con system prompt e skills dedicati
- State isolation tramite Redis con namespace per canale/gruppo
- Deploy come processi separati (o container Docker)

## Key Discoveries

- **Isolamento totale è la chiave**: i channel agents non devono MAI importare dal core — sono processi completamente autonomi
- **stateKey con namespace**: `wa:${groupId}` garantisce che ogni gruppo WhatsApp abbia il suo contesto conversazionale
- **Config YAML per canale**: permette system prompt, skills e permessi diversi per WhatsApp vs Telegram
- **Baileys per WhatsApp, grammY per Telegram**: due librerie diverse, due entry point, stesso pattern architetturale
- **Redis per stato**: Vector DB + Redis come state store permette persistenza e query rapide
- **Deploy indipendente**: ogni canale come processo separato → zero downtime cross-channel durante aggiornamenti
- **Skills dedicate per canale**: `channels/whatsapp/skills/` contiene solo le skills per WA, non quelle del core

## Open Questions

- Come condividere dati tra core e channel agents quando necessario? (event bus? shared Redis keys?)
- Possibilità di hot-reload delle config YAML senza riavviare il processo?
- Come gestire il fallback se un channel agent crasha? (supervisord? pm2?)
- Performance di Redis su 16GB RAM con molti gruppi attivi?
