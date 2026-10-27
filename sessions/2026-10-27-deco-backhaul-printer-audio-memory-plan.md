---
title: "Mini PC di casa: stampante che cade per colpa del backhaul Wi-Fi del Deco, doppia cassa via sdoppiatore, piano della memoria unificata per gli agenti"
date: "2026-10-27"
author: "kos-domus"
status: "ready"
tags: ["troubleshooting", "configuration", "memory", "multi-agent", "security"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (mini PC, Ryzen 5 7430U, 16 GB)"
  ide: "VS Code + Claude Code"
  model: "claude-opus-5-5"
---

## Objective
Chiudere tre fili aperti dalla sessione precedente: la stampante di rete che continuava a sparire, l'audio su due casse (Logitech + Bose SoundLink Mini) e il piano della memoria unificata per tutti gli agenti.

## Context
La stampante (Brother laser Wi-Fi), il mini PC e un nodo Deco (mesh TP-Link) sono nella stessa stanza, il garage. Il nodo del garage era ritenuto collegato via Ethernet. Il monitor `stampante-stato` (pycups + ipptool, timer ogni 2 min) scrive da questa sessione uno storico dei cambi di raggiungibilità in `~/.local/state/stampante-history.log`, per trovare lo schema dei distacchi.

## Steps Taken
### 1. Audio su due casse
Bluetooth integrato non supportato dal kernel → sdoppiatore jack stereo TRS sull'uscita AUX, Logitech e Bose (ingresso AUX) in parallelo. Test a volume medio con `wpctl set-volume @DEFAULT_AUDIO_SINK@ 0.5`.
**Result**: entrambe funzionano. La Bose sembrava muta: era il suo volume locale, non il mini PC.

### 2. Stampante: diagnosi via API locale del Deco
Interrogata la lista dei nodi del mesh (libreria Python `tplinkrouterc6u`, login con la password dell'account TP-Link): per ogni nodo c'è il tipo di collegamento verso il master. Il nodo del garage risultava su **backhaul wireless**, non cablato, nonostante il cavo arrivasse fisicamente.
- Test sul posto: guaina dei connettori troppo alta e cavo crimpato male → il Deco ripiegava in silenzio sul Wi-Fi, con un collegamento debole che si portava dietro tutti i client del garage (stampante compresa).
- Rakki ha rifatto il cavo.
**Result**: nodo garage su Ethernet a 1 Gbit, stampante di nuovo online e raggiungibile via IPP.

### 3. Piano della memoria unificata (solo progettazione)
Scritto un piano (skill `/plan`, modalità SOFTWARE) per una memoria comune agli agenti (Claude Code e i profili Hermes, più avanti un client desktop in sola lettura):
- **federare prima, consolidare dopo**: un indicizzatore SQLite FTS5 legge le memorie *dove sono già* (memorie di Claude, MEMORY.md dei profili Hermes, DOSSIER dei progetti, vault Obsidian);
- un server MCP stdio "memoria" con i **compartimenti fissati all'avvio** (casa, lavoro, personale, posta, condiviso): ogni agente vede solo i suoi; tool `search/get/recent/propose/supersede`;
- strategia di modifica per livello: fatti ADD-only con `superseded_by`, regole modificate sul posto, decisioni sostituite ma conservate;
- 8 unità: U0 contenimento immediato (le cartelle di famiglia escono dalle root MCP filesystem degli agenti di lavoro), U1 schema e mappa dei compartimenti, U2 indicizzatore con 20 domande d'oro (recall@5 ≥ 85%), U3 server, U4 collegamento per ruolo, U5 dedup, U6 irrigidimento con utenti unix per agente, U7 accesso remoto in sola lettura, per ultimo;
- revisione di sicurezza obbligatoria prima di U3, U6 e U7.
**Result**: piano in DRAFT, in attesa delle decisioni aperte. Niente ancora eseguito.

## Key Discoveries
- **Un Deco con il cavo fatto male non dà errore: passa al backhaul wireless.** Il sintomo sono i client di quel nodo che cadono a intermittenza. Il tipo di backhaul si legge dall'API locale o dall'app; non va dato per scontato perché "il cavo c'è".
- Prima di comprare hardware (un cavo USB dedicato per la stampante) conviene verificare il percorso di rete del nodo più vicino.
- Con lo sdoppiatore jack il volume del sistema è uno solo: le differenze tra le casse si regolano sulle casse.
- Su un indice di memoria piccolo e in italiano, BM25 (FTS5) ha battuto gli embedding locali piccoli in un test precedente (73% contro 46%): si parte da FTS5 e si misura prima di aggiungere embedding.
- Il primo passo di una memoria unificata non è la migrazione ma il **contenimento**: l'inventario aveva trovato agenti di lavoro che potevano leggere dati di famiglia attraverso le root del server MCP filesystem.

## Errors & Solutions
| Error | Cause | Solution |
|---|---|---|
| Stampante che sparisce dalla rete a intermittenza | nodo Deco del garage su backhaul wireless debole, cavo crimpato male | cavo rifatto, nodo su Ethernet |
| Bose muta con lo sdoppiatore | volume della cassa al minimo | alzato sulla cassa |

## Final State
Stampante e nodo Deco del garage su Ethernet; audio su due casse; piano della memoria unificata pronto in DRAFT.

## Open Questions
- Il cavo USB dedicato per la stampante serve ancora? Da decidere dopo una settimana di storico in `stampante-history.log`.
- Memoria unificata, decisioni aperte: store consolidato nel vault Obsidian? l'agente di casa vede parte del personale? scope dell'accesso remoto? un utente unix per agente? le memorie di Claude restano come indice personale?
