---
title: "OpenClaw Capture topic ACK ripristinato + scoperta hard limit 12000 chars su SOUL.md"
date: "2026-05-09"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "troubleshooting", "memory", "context"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Linux 6.17.0-22-generic (mini-PC kos-domus)"
  ide: "VS Code Remote (SSH)"
  model: "claude-opus-4-7[1m]"
---

## Objective

Ripristinare l'ACK "captured!" che Master Control inviava nel topic `📥 Capture` del supergroup Telegram Job-desk dopo aver archiviato un input (voice / text / link / document), e capire perché — pur dopo aver editato `SOUL.md` con la nuova istruzione — l'ACK non arrivava sul prossimo invio reale.

In parallelo: health-check dell'infrastruttura OpenClaw multi-agent (54 cron jobs + 5 workspace agent: orchestrator, cso, frontend-specialist, backend-expert, orchestration-architect) e fix collaterale del prompt cron `MC: Morning Todo List` che inviava al target alias name "Job-desk" invece del chatId numerico.

## Context

OpenClaw fleet: orchestrator (Master Control) coordina 4 specialist agent (CSO, Frontend, Backend, Orch.Arch) via Telegram supergroup forum `TG_CHAT_ID` con thread_id per topic. Capture è il topic `📥` di intake-only per quick capture di idee/foto/audio: l'orchestrator legge il topic, scarica il media, lancia la pipeline audio (Whisper locale < 2 min / AssemblyAI altrimenti) e archivia in `~/Obsidian-Personal/Inbox/<job_id>/` + crea nota indice `~/Obsidian-Personal/Inbox/YYYY-MM-DD-HHMM-<slug>.md` con frontmatter `source: telegram-capture`.

Storicamente Master Control inviava un ACK breve ("captured!") dopo l'archiviazione. Da qualche giorno l'utente non lo riceveva più: silenzio totale dopo il drop di un voice message in Capture. La suspect iniziale era una regressione del SOUL.md (sezione "SILENT BY DEFAULT" che istruiva il bot a non rispondere).

## Steps Taken

### 1. Health-check infrastruttura (read-only)
- `cat ~/.openclaw/cron/jobs.json` → 54 cron jobs totali, 7 by orchestrator (incluso Morning Todo, Evening Digest, Weekly Review). Tutti con `target` impostato.
- `ls ~/.openclaw/workspace-*/SOUL.md` → 5 file presenti (uno per agent), tutti coerenti come naming.
- Log gateway → no error visibili recenti, sessioni agent salutari.
- `wc -c ~/.openclaw/workspace-orchestrator/SOUL.md` → **14940 chars**. Primo flag visivo: il file è più lungo del solito.

### 2. Identificazione del topic Capture
Verifica delle session file `~/.openclaw/sessions/<...>-topic-80.jsonl` con trajectory contenente write a `~/Obsidian-Personal/Inbox/`. Conferma: thread_id `80` = Capture, thread_id `1` = Master Control. Cross-check via t.me link forniti dall'utente: `https://t.me/c/TG_CHAT_ID_SHORT/80` → Capture.

### 3. Restore ACK pattern in SOUL.md (fix #1)
Sostituita la sezione "SILENT BY DEFAULT" con "ACK ATTIVO" che istruisce esplicitamente:
- Pipeline media → archiviazione → invio messaggio breve "captured!" come ack nel topic 80.
- Tool message format esplicito: `target: "TG_CHAT_ID", thread_id: 80`.
- Pattern caratteristico: il framework auto-injecta sempre il chatId numerico via `target`, mai l'alias name.

Smoke test #1 con file audio dropped in Capture → **fallito**, nessun ACK ricevuto. Strano: la sezione era visibile nel SOUL.md.

### 4. Scoperta hard limit 12000 chars su SOUL.md (root cause)
Riapertura log subsystem `agent/embedded` durante una nuova trajectory:
```
{"subsystem":"agent/embedded"} workspace bootstrap file SOUL.md is 14940 chars
(limit 12000); truncating in injected context
(sessionKey=agent:orchestrator:telegram:group:TG_CHAT_ID:topic:80)
```

Il framework tronca silenziosamente i file di bootstrap workspace (`SOUL.md` + `MEMORY.md` + `USER.md` + `AGENTS.md` + `TOOLS.md`) quando superano i 12000 chars per file. La sezione "ACK ATTIVO" che avevo aggiunto era in coda al SOUL → finiva oltre il limite → invisibile al modello a runtime. Il modello continuava a seguire il content delle prime ~12000 chars (snapshot di una iterazione precedente con SILENT BY DEFAULT ancora vigente).

### 5. Trim SOUL.md sotto soglia
Strategia: estrarre i dettagli operativi del Capture flow in un file separato `procedures/capture-protocol.md`, lasciando nel SOUL le **regole essenziali** + l'**ACK pattern** corti. Pattern già usato per altre procedure (audio-and-meet, youtube, project-dossier).

Operazioni:
- Creato `~/.openclaw/workspace-orchestrator/procedures/capture-protocol.md` con i dettagli per voice / text / document handling (frontmatter Obsidian, slug rule, materiali wikilink, vincoli operativi).
- Trim di sezioni verbose nel SOUL: "Opinions" e "Miglioramento Continuo" e "Lookup discipline" compressed.
- Final: `wc -c ~/.openclaw/workspace-orchestrator/SOUL.md` → **11421 chars** (con buffer ~600 chars sotto il limit).

Smoke test #2 con file audio dropped in Capture → **successo**: ACK "captured!" ricevuto nel topic 80, file audio archiviato in `~/Obsidian-Personal/Inbox/<job_id>/`, nota indice creata. Pipeline confermata end-to-end.

### 6. Fix collaterale: prompt cron `MC: Morning Todo List`
Durante l'health-check, scoperto che STEP 5 del prompt (sezione "Pubblica su Telegram nel gruppo Job-desk") faceva sì che MC tentasse di usare l'alias name `"Job-desk"` come `target`, restituendo `Unknown target "Job-desk"` perché il framework si aspetta solo chatId numerici.

Fix: patchato STEP 5 in `~/.openclaw/cron/jobs.json` con istruzione esplicita "delivery via framework auto-routing — nessun tool message manuale necessario" + caveat per i casi in cui MC volesse comunque inviare ("usa SEMPRE chatId numerico `target: \"TG_CHAT_ID\"` e mai alias name").

Backup: `jobs.json.before-job-desk-fix-2026-05-09`.

### 7. MEMORY.md orchestrator esteso
Aggiunta tabella esplicita topic IDs + "Tool message — uso corretto" con esempio canonico `target: "TG_CHAT_ID", thread_id: <N>` per ognuno dei 5 topic. Caveat: senza `thread_id` il messaggio finisce nel topic General = Master Control by default in supergroup forum.

## Configuration Changes

- `~/.openclaw/workspace-orchestrator/SOUL.md`: 14940 → 11421 chars. Sezione "SILENT BY DEFAULT" rimossa. Aggiunta sezione "ACK ATTIVO" con tool message example. Trim di sezioni verbose. Backup: `SOUL.md.before-capture-ack-fix-2026-05-09`.
- `~/.openclaw/workspace-orchestrator/procedures/capture-protocol.md`: NUOVO file (~3KB) con flow dettagliato voice/text/document, slug rule, materiali wikilink, vincoli pipeline.
- `~/.openclaw/workspace-orchestrator/MEMORY.md`: aggiunta tabella topic IDs + tool message format esplicito.
- `~/.openclaw/cron/jobs.json`: STEP 5 di `MC: Morning Todo List` riscritto. Backup: `jobs.json.before-job-desk-fix-2026-05-09`.

## Key Discoveries

- **Hard limit silenzioso 12000 chars su file workspace bootstrap**: il framework `agent/embedded` tronca senza errore a runtime quando un file di bootstrap (SOUL/MEMORY/USER/AGENTS/TOOLS) supera i 12000 chars. Il warning è solo nei log a livello info, non viene mai ritornato all'utente o all'agent. Sintomo: feature documentate **al fondo** del file non funzionano, mentre feature in alto continuano a comportarsi normalmente.
- **Pattern di estrazione `procedures/<topic>.md`**: è il workaround standard quando una sezione del SOUL cresce oltre il budget. Il SOUL referenzia (`vedi procedures/<topic>.md`), il modello carica il file on-demand via Read tool quando il topic è attivo. Pattern già usato in workspace-orchestrator per audio-and-meet, youtube, project-dossier; aggiunto capture-protocol.md.
- **`target` Telegram supergroup forum**: SEMPRE chatId numerico (`TG_CHAT_ID`), mai alias name. Senza `thread_id` il messaggio finisce in General = topic 1 = Master Control. Caveat documentato nel MEMORY.md dell'orchestrator perché ricorrente.
- **Visual swap Telegram (Master Control ↔ Capture sidebar)**: glitch rendering del client Telegram durante session attive. Non è un bug del framework — i topic IDs sono confermati via t.me link diretti.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| ACK "captured!" non arriva nel topic Capture nonostante SOUL.md aggiornato | SOUL.md a 14940 chars > hard limit 12000, sezione ACK al fondo veniva troncata silenziosamente nel context iniettato all'agent | Estratto Capture flow dettagliato in `procedures/capture-protocol.md`, trim sezioni verbose. SOUL → 11421 chars con buffer |
| Cron `MC: Morning Todo List` log: `Unknown target "Job-desk"` | Prompt step 5 diceva "Pubblica su Telegram nel gruppo Job-desk" → MC interpretava "Job-desk" come `target` invece del chatId numerico | Patchato step 5: delivery via framework auto-routing + caveat sempre chatId numerico |

## Final State

- Capture topic operativo: ACK "captured!" inviato dopo ogni archiviazione, pipeline audio → Obsidian Inbox + nota indice funzionante end-to-end. Smoke test confermato con voice message reale.
- MC Morning Todo cron: prompt step 5 ripulito, target alias eliminato.
- SOUL.md orchestrator a 11421 chars con buffer sotto il limit. Pattern `procedures/<topic>.md` adottato per Capture come per altri flow on-demand.
- MEMORY.md orchestrator esteso con tabella topic IDs e tool message format.
- Health-check infrastruttura: 54 cron jobs validati read-only, nessun altro flag.

## Open Questions

- Il limit 12000 chars è hardcoded in `agent/embedded` o configurabile? Per ora trattato come hard constraint via memory `feedback_openclaw_soul_size_limit.md`.
- Oltre al SOUL, gli altri file di bootstrap (MEMORY/USER/AGENTS/TOOLS) sono ognuno sotto soglia o cumulativi? Per ora regola applicata per-file (target ~11000 chars per buffer).
- Sarebbe utile uno script `~/.openclaw/scripts/check-bootstrap-size.sh` che faccia `wc -c` su tutti i workspace e segnali file > 11000 chars come warning. Deferred — si fa al primo prossimo workaround.
