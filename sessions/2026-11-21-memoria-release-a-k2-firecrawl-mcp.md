---
title: "Release A della memoria, password fuori dagli argomenti, Firecrawl come MCP remoto con allowlist"
date: "2026-11-21"
author: "kos-domus"
status: "processed"
tags: ["memory", "security", "mcp", "mcp-servers", "hooks", "settings", "automation", "multi-agent"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (mini PC), kernel 7.0"
  ide: "VS Code + Claude Code"
  model: "Opus 5.5 (orchestratore), Fable per il workflow di pianificazione"
---

## Objective
Chiudere la settimana: installare la release A della memoria unificata, chiudere condizioni di sicurezza aperte (password del
DB visibile nei processi, file temporanei leggibili da tutti), rimettere Firecrawl in Claude Code in modo sicuro, aggiungere
Reddit al report settimanale di GitHub Trending dell'agente Kos.

## Context
Mini PC con una flotta di agenti (Hermes) e Claude Code. Le decisioni di sicurezza passano da un subagente CSO; i piani grossi
da un co-CEO e da una revisione cross-provider. Le operazioni irreversibili le lancia Rakki da un utente amministrativo
separato.

## Steps Taken
### 1. Release A della memoria (installazione guidata, ~25 minuti)
Go/no-go con tutte le verifiche rilanciate sul codice (8/8 verdi, impronte degli script invariate). Installazione da una copia
congelata posseduta da root, controllata con lo sha256 del manifest comunicato in chat (non letto dal file scrivibile),
copia della release precedente per il ritorno, `cmp` dopo l'install, rebuild, verifiche di nuovo verdi. Misura «prima/dopo» su
10 domande con giudice cieco: nessun peggioramento.
**Result**: installata. Prima dell'install Rakki ha confermato i titoli delle regole che gli agenti vedranno come «approvate».

### 2. Password del DB fuori dagli argomenti
Quattro script passavano l'URL con password a `psql`/`pg_dump`: chiunque sul sistema lo vede con `ps`. Helper comune che
ricava un URL senza password ed esporta la password solo in `PGPASSWORD` (l'ambiente di un processo lo legge solo lo stesso
utente o root). Verifica: `psql` lento osservato con `/proc/<pid>/cmdline`. Tolto un server MCP postgres non usato (lanciato
con `npx` senza versione e con la password nella config). Rotazione con `\password` (la password viaggia già cifrata) e prova
che un altro utente non entri e non legga l'ambiente.

### 3. Firecrawl come MCP remoto con allowlist
Il vecchio server MCP puntava a uno script sparito. Parere CSO: no a `firecrawl setup mcp` (esegue un pacchetto di terzi con
`npx` e configura anche altri agenti), no al pacchetto locale (legge il `.env` della cartella corrente: un repo potrebbe
dirottare la chiave), sì al server remoto ufficiale aggiunto a mano con la chiave come riferimento `${VAR:-}` nell'header.
Hook `PreToolUse` che ammette solo scrape, search, map, credit_usage e developer_search (con limiti sui parametri); tutto il
resto, compresi strumenti nuovi aggiunti dal fornitore, è bloccato. Controllo settimanale che URL, header e hook non cambino.
**Result**: dopo il riavvio il server esponeva strumenti nuovi non visti dal CSO: bloccati per default, come previsto.

### 4. Reddit nel report di GitHub Trending (idea presa da last30days-skill)
Le API `.json` di Reddit dai cron sono bloccate. Script nostro: ricerca web `site:reddit.com "<repo>"` limitata all'ultimo mese
via API ufficiale di Firecrawl, solo link reddit.com, estratti tagliati, filtro per nomi comuni; il testo di Reddit è dato, mai
istruzione.

### 5. Altro
Build della Mini App di SpesaBot da elenco chiuso di file (logo da 3,9 MB a 153 KB) e API con ambiente minimo (9 variabili
invece di ~50). Pulizia di offerte doppie nate da un giro di scraping che carica la settimana dopo nel volantino corrente.
`/print` per stampare l'ultimo allegato da Telegram e WhatsApp. Aggiornamento di sistema e riavvio.

## Configuration Changes
- `~/.claude/settings.json`: hook `PreToolUse` per `mcp__firecrawl__.*` + `permissions.deny` di riserva.
- Server MCP `firecrawl` (scope user) remoto HTTP con header `Bearer ${FIRECRAWL_API_KEY:-}`.
- Helper `pg-senza-password.sh` nei 4 script del DB; MCP postgres rimosso da due profili Hermes e dal `.mcp.json` del workspace.
- Drop-in systemd per l'ambiente minimo dell'API (`exec env -i …`).

## Key Discoveries
- **Un PDF «oscurato» col pennarello nero non è oscurato**: il testo resta nel livello di testo. Appiattire in immagine o usare
  dati inventati.
- **La verifica «prima/dopo» ha scoperto un problema vero**: le impronte dei file generati «dopo» coincidevano con una misura di
  due giorni prima → file di produzione scritti in precedenza da codice non installato (probabilmente esecutori automatici che
  potevano lanciare Bash). Le misure con impronta degli input pagano.
- La password in chiaro trovata nella config di due agenti era **vecchia** (rifiutata dal DB): esposta, ma già morta. Controllare
  sempre se un segreto trovato è ancora valido prima di stimare il danno.
- `pgrep -fc pattern` conta anche la shell che contiene il pattern nella riga di comando.
- Un server MCP remoto cambia strumenti senza preavviso: l'allowlist via hook è l'unico punto fermo lato client.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| MCP firecrawl `ENOENT` | wrapper in una cartella rimossa | server remoto ufficiale + allowlist |
| Ricerca su disco della password oltre il timeout | python che legge tutto il disco | `grep -rlF -f file_0600` con esclusioni |
| Controllo del dump non osservato | il servizio finisce in 3 s | prova diretta con `psql` e `pg_sleep` |

## Final State
Release A installata e verificata; condizioni di sicurezza K1/K2 chiuse (K3 pronta per l'amministratore); Firecrawl MCP con
allowlist provata dal vivo; Reddit nel report del lunedì.

## Open Questions
- Correggere l'esecutore automatico dei piani perché non possa lanciare Bash né scrivere file di produzione.
- Umask dei file temporanei degli agenti (oggi leggibili da tutti a ogni nuovo report).
