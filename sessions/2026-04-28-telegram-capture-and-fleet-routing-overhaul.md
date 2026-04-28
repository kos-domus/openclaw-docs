---
title: "Telegram Capture topic, MC redirect to work group, weekly Inbox triage cron, agent identifying prefixes, per-agent Obsidian vault access"
date: "2026-04-28"
author: "kos-domus"
status: "ready"
tags: ["multi-agent", "configuration", "automation", "cron", "agent-sdk", "memory", "context", "scheduling"]
openclaw_version: "2026.4.24"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "Claude Code (CLI + VSCode)"
  model: "claude-opus-4-7[1m]"
---

## Objective

Razionalizzare la fleet (Kos / Kai / Master Control / Mr Wolf) sui due assi mancanti dopo il setup Obsidian del giorno prima:
1. **Conoscenza condivisa** — ogni agente deve poter consultare il vault Obsidian con permessi appropriati al proprio ruolo, senza centralizzare tutto su un singolo agente.
2. **Telegram bidirezionale** — trasformare il "Job Desk" Telegram (group multi-topic) da feed di notifiche push-only a knowledge capture layer, mantenendo costo basso (~€2/mese vs €15-60/mese di full-interactive).

Side goal: chiarire visivamente CHI sta scrivendo nel feed Telegram condiviso (MC e Kos pubblicano entrambi sullo stesso chat).

## Context

**Stato di partenza**:
- 4 agenti principali, ognuno col proprio SOUL.md ma nessuno con accesso esplicito al vault Obsidian a `~/Obsidian-Personal/`.
- Vault già attivo e popolato (Daily/, Decisions/, Inbox/, Excalidraw/), con MC che scrive Daily Briefing 06:30 nel vault.
- Tutti i cron Telegram-bound (sia di MC che di Kos) facevano delivery sullo STESSO chat personale (`TG_USER_ID`), causando una percezione errata: "tutti i report sembrano venire da Kos".
- Telegram work group multi-topic `TG_WORK_GROUP_ID` configurato con bindings per orchestrator + 4 specialisti (CSO/Frontend/Backend/OrchArch su topic 3-6), ma **sotto-utilizzato** — i cron MC pubblicavano sul personal chat invece che nel group.
- Audio pipeline preesistente in `~/job-desk/family-os/99_kai_system/scripts/audio-pipeline/` con supporto multi-agent (`--agent kai|master|wolf`).

**Vincoli**:
- API Anthropic resta deep fallback only (memoria `feedback_anthropic_api_cost`).
- File `openclaw.json` viene riscritto dal gateway al graceful shutdown — edit a runtime rischia clobber. Procedura corretta: stop gateway → edit → start.
- SOUL.md ha limite hard di 12000 caratteri prima del troncamento del bootstrap context.

## Steps Taken

### 1. Per-agent Obsidian vault access (4 SOUL.md updates)

Aggiunta sezione "Obsidian Vault" con permessi calibrati al ruolo:

| Agente | Path SOUL | Access | Frame |
|---|---|---|---|
| **Master Control** | `workspace-orchestrator/SOUL.md` | read+write (canonical owner) | Hub primario. Daily/Decisions/Inbox. Scrive Daily Briefing 06:30 sia in vault che Drive. |
| **Kos** | `workspace-cos/SOUL.md` | read only | Per CoS work (context fleet/strategic). Se emerge una decisione, segnala a MC che la scrive. |
| **Kai** | `workspace-family/SOUL.md` | read limitato (solo Daily/) | Privacy boundary esplicito: il vault è di Rakki, non Family. Mai inoltrare contenuti vault nel gruppo Family senza filtro. |
| **Mr Wolf** | `workspace-wip/SOUL.md` | read only (Decisions + Daily) | Per context architetturale cross-progetto. Le decisioni WIP vivono nel suo MEMORY/SCOPE_EVOLUTION, non nel vault. |

**Result**: ciascun agente sa ora dove guardare e cosa può/non può fare nel vault. Master Control resta il canonical writer (vincolo dell'utente: "Kos ha altre funzioni, è MC che deve occuparsi di questo").

### 2. Communication prefix per distinguere MC vs Kos nel feed Telegram

Aggiunta sezione "Communication prefix (Telegram)" in tutti e 4 i SOUL:

| Agente | Prefisso |
|---|---|
| Master Control | `🎯 MC —` |
| Kos | `📚 Kos —` |
| Kai | `👨‍👩‍👧 Kai —` (solo cross-post; in Family WhatsApp non serve) |
| Mr Wolf | `🐺 Wolf —` |

Implementato a livello SOUL invece che cron payload → si applica a TUTTI i messaggi Telegram dell'agente (sia da cron che da sessione interattiva).

**Result**: dal prossimo cron run, ogni messaggio è identificabile a colpo d'occhio nel feed.

### 3. MC briefing/digest → work group (config change)

Modifica `~/.openclaw/cron/jobs.json`: per i due cron MC (`Daily Work Briefing` 06:30, `Evening Digest` 19:30), cambiato `delivery.to` da `TG_USER_ID` (personal chat) a `TG_WORK_GROUP_ID` (work group, general topic dove MC è già bound).

Backup pre-modifica: `~/.openclaw/cron/jobs.json.before-mc-redirect-2026-04-28`.

Editato a runtime senza fermare il gateway — `jobs.json` non ha il problema di clobber che ha `openclaw.json` (nessun warning nei log post-edit, prossimo schedule tick funzionante).

**Result**: dal prossimo Evening Digest 19:30, briefing arrivano nel work group invece che nel personal chat. Personal chat `TG_USER_ID` resta per intelligence reports di Kos (GitHub Trending, API Cost) e ha quindi una funzione semantica diversa.

### 4. Audio pipeline esteso con `--agent capture`

Modificato `~/job-desk/family-os/99_kai_system/scripts/audio-pipeline/config.py`:
```python
AGENT_OUTPUT_DIRS = {
    "kai":     ...,
    "master":  ...,
    "wolf":    ...,
    "capture": Path.home() / "Obsidian-Personal" / "Inbox",  # NEW
}
AGENT_DRIVE_FOLDERS = {
    ...,
    "capture": None,  # vault-only, no Drive copy
}
```

**Result**: qualsiasi agente può ora chiamare `run.sh /tmp/voice.ogg --agent capture --name "<slug>"` e l'output finisce direttamente nel vault Inbox/.

### 5. Telegram Capture topic setup + binding

User ha creato il topic `📥 Capture` nel work group e fornito link `t.me/c/3701546472/80` → topic ID = `80`, peer completo `TG_WORK_GROUP_ID:topic:80`.

CLI `openclaw agents bind --bind <channel:accountId>` NON supporta peer-scoped binding (syntax `channel[:accountId]` solamente). Tentativi di passaggio peer come parte della stringa `--bind` aggiungono binding NON-scoped (cattura tutto telegram:work, troppo broad).

Procedura adottata:
1. `cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.before-capture-binding-2026-04-28`
2. `systemctl --user stop openclaw-gateway`
3. Append diretto a `bindings[]`:
   ```json
   {
     "type": "route",
     "agentId": "orchestrator",
     "match": {
       "channel": "telegram",
       "accountId": "work",
       "peer": { "kind": "group", "id": "TG_WORK_GROUP_ID:topic:80" }
     }
   }
   ```
4. `systemctl --user start openclaw-gateway`
5. Verifica: `openclaw agents bindings | grep orchestrator` → mostra entrambe le bindings (general + topic:80) ✓

**Result**: messaggi nel topic 80 vengono routati a Master Control.

### 6. Capture Protocol in MC SOUL.md

Aggiunta sezione "Capture Protocol — knowledge intake from Telegram" che definisce:
- **Voice flow**: scarica audio → `run.sh --agent capture` → executive_summary in `Inbox/<job_id>/` → crea nota indice in `Inbox/YYYY-MM-DD-HHMM-<slug>.md` con frontmatter + link Obsidian wikilink ai materiali.
- **Text flow**: scrivi direttamente in `Inbox/YYYY-MM-DD-HHMM-<slug>.md`.
- **Conferma**: 1 riga su Telegram con prefisso `🎯 MC —`.
- **Vincoli**: NESSUNA analisi/sintesi extra, è un INBOX (lo triagia Rakki dopo). Slug = first 4-6 parole kebab-case.

**Wikilink format strutturato** (refinement post-test):
```markdown
## Materiali
- [[Inbox/<job_id>/transcript|Full transcript]]
- [[Inbox/<job_id>/executive_summary|Executive summary (full)]]
- [[Inbox/<job_id>/detailed_summary|Detailed summary]]
```
Per audio < 500 parole, transcript inline + sezione Materiali con un solo link. Per audio lunghi, executive_summary inline + 3 link al canonical content.

### 7. Bug discovery: SOUL.md truncation a 12000 chars

Primo test in topic 80 → "nessun output". Nei log gateway:
```
workspace bootstrap file SOUL.md is 12572 chars (limit 12000); truncating in injected context (sessionKey=agent:orchestrator:telegram:group:TG_WORK_GROUP_ID:topic:80)
```

Il routing aveva funzionato (il sessionKey contiene topic:80) ma il bootstrap del SOUL veniva troncato — la sezione Capture Protocol probabilmente cadeva nel troncamento.

**Fix**: split delle sezioni meno usate in `procedures/`:
- `~/.openclaw/workspace-orchestrator/procedures/audio-and-meet.md` (Audio Intelligence + Google Meet, ~2K chars)
- `~/.openclaw/workspace-orchestrator/procedures/youtube.md` (YouTube/Podcast Intelligence, ~600 chars)

SOUL aggiornato con sezione "Riferimenti operativi" che linka le procedure on-demand. Risultato: SOUL passa da 12572 → 10396 chars (margine sicuro per future estensioni).

**Result**: prossimo messaggio nel topic Capture → MC ha istruzioni complete → archivia in vault Inbox/ → conferma con `🎯 MC —`. Verificato dall'utente ("L'ha registrato!").

### 8. Inbox Triage weekly cron (Kos, lun 09:00)

Aggiunta sezione "Inbox Triage settimanale" in `workspace-cos/SOUL.md`. Definisce 4 categorie di classificazione (Decision-candidate / Follow-up / Idea / Discard), schema report con prefisso `📚 Kos —`, vincoli (no movimento file, solo suggerimento).

Cron job aggiunto a `jobs.json`:
```json
{
  "id": "<uuid>",
  "agentId": "cos",
  "name": "Inbox Triage settimanale",
  "enabled": true,
  "schedule": { "kind": "cron", "expr": "0 9 * * 1", "tz": "Europe/Rome" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Esegui l'Inbox Triage settimanale come definito nel tuo SOUL.md..."
  },
  "delivery": { "mode": "announce", "channel": "telegram", "to": "TG_USER_ID" }
}
```

Backup: `~/.openclaw/cron/jobs.json.before-inbox-triage-2026-04-28`.

**Result**: ogni lunedì 09:00 Europe/Rome, Kos scansiona Inbox/ degli ultimi 7 giorni, classifica e suggerisce triage via Telegram personal chat. Force review periodico per evitare che Inbox/ diventi un cimitero di pensieri morti.

## Configuration Changes

| File | Change |
|---|---|
| `~/.openclaw/workspace-orchestrator/SOUL.md` | + "Obsidian Vault" + "Communication prefix" + "Capture Protocol" + sostituzione Audio Intelligence/Meet/YouTube con riferimenti a `procedures/`. Net: 12572 → 10984 chars |
| `~/.openclaw/workspace-orchestrator/procedures/audio-and-meet.md` | NUOVO (procedure on-demand) |
| `~/.openclaw/workspace-orchestrator/procedures/youtube.md` | NUOVO (procedure on-demand) |
| `~/.openclaw/workspace-cos/SOUL.md` | + "Communication prefix" + "Inbox Triage settimanale" + "Obsidian Vault" |
| `~/.openclaw/workspace-family/SOUL.md` | + "Communication prefix" + "Obsidian Vault" (read limitato + privacy boundary) |
| `~/.openclaw/workspace-wip/SOUL.md` | + "Communication prefix" + "Obsidian Vault" |
| `~/.openclaw/openclaw.json` | + binding `orchestrator → -1003...:topic:80` (Capture topic) |
| `~/.openclaw/cron/jobs.json` | MC briefings: `delivery.to` redirect TG_USER_ID → TG_WORK_GROUP_ID; + Inbox Triage cron lun 09:00 |
| `~/job-desk/family-os/99_kai_system/scripts/audio-pipeline/config.py` | + `capture` agent → vault Inbox/ |

Backup files in `~/.openclaw/`:
- `openclaw.json.before-capture-binding-2026-04-28`
- `cron/jobs.json.before-mc-redirect-2026-04-28`
- `cron/jobs.json.before-inbox-triage-2026-04-28`

## Key Discoveries

- **`openclaw agents bind --bind` non supporta peer scope**: il flag accetta solo `<channel[:accountId]>`, niente sintassi peer/topic. Per topic-scoped routing serve edit diretto a `openclaw.json` con stop+start del gateway. Le bindings esistenti (CSO/FE/BE/OrchArch su topic 3-6) erano state aggiunte con questa procedura prima.
- **SOUL.md hard limit 12000 chars**: oltre, il bootstrap viene troncato silenziosamente (warning nei log ma nessun errore visibile all'utente). Strategia di mitigation: split sezioni occasionali in `procedures/<topic>.md` caricabili on-demand. Pattern già usato in workspace-family.
- **`openclaw.json` clobber on graceful shutdown**: il gateway re-serializza il config in-memory. Edit a runtime → perso al prossimo restart. `cron/jobs.json` invece è safe da editare a runtime (cron service legge ad ogni tick).
- **Telegram peer ID format per topic**: `<group_id>:topic:<thread_id>` dove group_id ha prefisso `-100` per supergroup. Topic ID si recupera da link Telegram `t.me/c/<group_internal>/<topic_id>` o forwardando un msg a `@RawDataBot`.
- **Cron sessions sono isolated**: ogni run ricarica fresh SOUL.md da disco, niente da riavviare dopo edit del SOUL. Stesso vale per inbound Telegram routing — ogni nuova session legge il SOUL corrente.
- **Audio pipeline `AGENT_DRIVE_FOLDERS` non è ancora wired**: il dict esiste in config.py ma `pipeline.py` non lo usa per upload Drive automatico. Settato `None` per agent `capture` ma è solo metadata.
- **Visual identification del feed Telegram condiviso**: prefisso emoji+nome è sufficiente, niente bisogno di chat separati per agente. Pattern: `<emoji> <NomeAgente> —` come prima riga di ogni messaggio outbound.
- **Privacy boundary cross-domain (Kai)**: il vault è personale di Rakki, ma Kai opera nel context Family (Rakki+Katia). Read access solo a Daily/ + regola esplicita "filtra prima di postare in Family WhatsApp" — distinguere context personale da context family è critico.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `openclaw agents bind --bind 'telegram:work:group:...:topic:80'` aggiunge binding troppo broad (no peer scope) | CLI `--bind` accetta solo `channel[:accountId]`, ignora il resto | `unbind` la versione broad, edit diretto `openclaw.json` con stop+start gateway |
| `nessun output` su test capture in topic 80 | SOUL.md di MC = 12572 chars, oltre limit 12000 → bootstrap troncato silenziosamente | Split Audio Intelligence/Meet/YouTube in `procedures/`, SOUL scende a 10396 chars |
| Bindings ID file ricostruiti sha256 ad ogni edit (`changedPaths=56`) | Normale: il diff è calcolato su tutto il file, non solo i path realmente cambiati | Non è un errore, solo log verboso |

## Final State

**Capture flow end-to-end live**: 
1. Rakki droppa voce/testo nel topic 📥 Capture (TG_WORK_GROUP_ID:topic:80) →
2. Gateway routa a Master Control via binding peer-scoped →
3. MC esegue Capture Protocol: trascrive (audio pipeline `--agent capture` se voce), archivia in `~/Obsidian-Personal/Inbox/YYYY-MM-DD-HHMM-<slug>.md` con frontmatter + wikilink ai materiali full →
4. Conferma su Telegram con prefisso `🎯 MC —` →
5. Syncthing replica al Mac entro pochi secondi.

**MC briefings nel work group**: Daily Briefing 06:30 + Evening Digest 19:30 ora in `TG_WORK_GROUP_ID` (general topic) invece che `TG_USER_ID`. Personal chat resta per intelligence reports di Kos.

**Inbox Triage automatico**: ogni lunedì 09:00, Kos scansiona Inbox/ ultima settimana, classifica in 4 categorie, manda report con prefisso `📚 Kos —`. Niente movimento file automatico — solo suggerimento, Rakki decide.

**Identificazione visuale feed**: ogni agente posta col proprio prefisso (🎯 MC, 📚 Kos, 🐺 Wolf, 👨‍👩‍👧 Kai) → no più ambiguità su chi sta scrivendo.

**Costo stimato pipeline Capture**: Whisper locale per audio < 2 min (gratis), AssemblyAI per più lunghi (~0.6¢/min) + 1 turn breve di MC (~3-5¢ via Anthropic, meno con altri provider). 10 capture/giorno = ~€2/mese.

## Open Questions

- **Triage UX**: il primo run di Inbox Triage (lun prossimo) sarà il vero test — se Rakki trova il report troppo verboso o troppo asciutto, iterare lo schema nel SOUL Kos.
- **Drive sync per Capture**: `AGENT_DRIVE_FOLDERS["capture"] = None` (vault-only). Se servisse mirror su Drive (per backup cloud, accesso da altri dispositivi senza Obsidian Sync), `pipeline.py` non legge ancora questo dict — feature gap.
- **Cleanup audio originali**: la pipeline scarta file processati dopo 7 giorni in `/tmp/audio-pipeline/processed/`. Nessuna verifica di policy retention sul vault stesso (le subfolder `Inbox/<job_id>/` accumulano per sempre). Potenziale follow-up: cleanup retention policy via Kos cron.
- **CLI `bind` con peer**: vale la pena aprire una issue/PR upstream per estendere `openclaw agents bind` con peer-scope? Pattern di binding peer-scoped è già diffuso (CSO/FE/BE/OrchArch + ora Capture), CLI dovrebbe supportarlo nativo.
- **Topic separato per "Briefings & Digest"** (proposta dell'utente, NON implementata): attualmente MC posta nel general topic del work group. Se il general diventa rumoroso, valutare topic dedicato per briefing/digest e tenere il general per chat libera.
