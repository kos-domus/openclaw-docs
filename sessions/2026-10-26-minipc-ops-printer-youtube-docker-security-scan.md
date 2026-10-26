---
title: "Mini PC di casa: CUPS che annullava i lavori, player YouTube senza ads per agenti, update Docker (HA/Immich/n8n), controllo di sicurezza settimanale"
date: "2026-10-26"
author: "kos-domus"
status: "ready"
tags: ["automation", "security", "cron", "troubleshooting", "configuration", "setup"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (mini PC, Ryzen 5 7430U, 16 GB)"
  ide: "VS Code + Claude Code"
  model: "claude-opus-5-5"
---

## Objective
Una serie di interventi di manutenzione e automazione sul mini PC che fa da server di casa (13–26/10): stampante, voce, media, aggiornamenti dei container e un controllo di sicurezza ricorrente.

## Steps Taken
### 1. Stampante di rete: lavori "spariti"
La stampante Wi-Fi si era scollegata dalla rete. Al rientro i lavori in coda risultavano `canceled / job-canceled-by-user` esattamente **3 ore** dopo la creazione: è il `MaxJobTime` di default di CUPS (10800 s).
**Fix**: `MaxJobTime 604800` (7 giorni) in `cupsd.conf`, verificato con un job in hold (`job-cancel-after = 604800`). Monitor `stampante-stato` (pycups + ipptool, timer ogni 2 min) che manda avvisi Telegram quando cambia lo stato: offline con lavori in attesa, carta, inceppamento, toner.

### 2. YouTube senza pubblicità per gli agenti
- `yt-player` = mpv + yt-dlp **in streaming** (niente su disco), comandato via socket IPC (play/pausa/volume/prossimo/stato, modalità `--video` sul monitor). yt-dlp installato con `uv tool` (la versione apt è troppo vecchia) con `--js-runtimes node`, più un timer settimanale `uv tool upgrade`.
- Firefox snap con **uBlock Origin forzato via `/etc/firefox/policies/policies.json`** (`ExtensionSettings.force_installed`) e `Permissions.Autoplay.Default = allow-audio-video`.
- Test silenziosi con `MPV_HOME` che punta a un `mpv.conf` con `ao=null`/`vo=null`.
- Il Bluetooth integrato (MediaTek **MT7902**) non è supportato dal kernel ("Unsupported hardware variant (00007902)"): per una cassa BT serve una chiavetta USB (chip Realtek RTL8761BU), oppure jack + sdoppiatore stereo TRS.

### 3. Aggiornamento dei container
- Immich 3.3.0 → 3.3.5 (release di sicurezza: CA eMudhra in certifi). Dump del DB prima dell'update; migrazione in 1 s.
- n8n 2.23.1 → 2.44.6, **fissato per digest**. Gotcha: dall'host `/var/lib/docker/volumes/<vol>/_data` sembrava vuoto; il backup a freddo è stato fatto da un container (`docker run --rm -v vol:/d:ro -v $B:/out alpine tar …`). Execute Command è disattivato di default da n8n 2.0.
- Home Assistant 2026.5.4 → 2026.10.2. Immagine vecchia ritaggata per il rollback, backup della config senza log. Breaking changes verificati sulla config reale (nessuno applicabile). Lo spam di log `kasa.smart.smartdevice` (~300k errori in 6 giorni) è sparito con la versione nuova.

### 4. Controllo di sicurezza settimanale (`security-weekly`)
Script deterministico (niente LLM, nessun auto-update):
- inventario (apt, snap, npm -g, uv/pipx, venv degli agenti, immagini Docker, estensioni Firefox, componenti HA) con diff settimanale;
- Trivy (aggiornato con verifica sha256 sul `checksums.txt` della release) su: un **rootfs minimo** (solo `os-release` + `dpkg/status`, così non scansiona i dischi dati), le cartelle dei pacchetti e le immagini Docker. Solo HIGH/CRITICAL con fix;
- **OSV querybatch** sui pacchetti npm/PyPI cercando gli ID `MAL-*` (pacchetti malevoli), validato con MAL-2022-1;
- esposizione: `ss` raggruppato per porta e servizio, esclusi loopback, reti Docker e porte effimere ≥ 32768 (senza filtro: 61 "servizi", dopo: 19 reali);
- report la domenica su Telegram, allarmi giornalieri solo se c'è qualcosa di nuovo.
Primo giro: 0 malevoli, 231 vulnerabilità HIGH/CRITICAL correggibili, raggruppate per "chi può correggere" (noi / progetti / fornitore). Corrette subito quelle "nostre" (uv/pipx, kernel vecchi) e quelle dei progetti (Fastify/@fastify/static in un'API esposta, certifi/aiohttp in ambienti Python).

## Key Discoveries
- CUPS annulla in silenzio i job dopo 3 ore se la stampante è irraggiungibile.
- Gli ambienti Python da scansionare vanno ricavati anche dall'`ExecStart` dei servizi systemd, non solo dalle workdir.
- Ricompilare un progetto TypeScript per un bump di dipendenze può portare in produzione modifiche WIP non compilate in `src/`: se `dist/` carica le dipendenze a runtime, basta il riavvio (controllo dei tipi con `tsc --noEmit`).
- Nei test di isolamento non usare mai `cat` sui file segreti: usare `test -r` o `stat`.

## Errors & Solutions
| Error | Cause | Solution |
|---|---|---|
| Lavori di stampa annullati "by user" | `MaxJobTime` di default (3 h) | `MaxJobTime 604800` |
| Cloudflare speed test 403 | `__down?bytes` oltre 50 MB | 50 MB massimo |
| Archivio del volume n8n vuoto | percorso host del volume non rappresentativo | tar da un container che monta il volume |
| Trivy con decine di "servizi esposti" | porte effimere e reti Docker contate | filtro e raggruppamento |

## Final State
Stampa affidabile con avvisi; player YouTube comandabile dagli agenti; HA, Immich e n8n aggiornati con rollback pronto; scansione settimanale attiva.

## Open Questions
- Chiavetta Bluetooth o sdoppiatore jack per la cassa.
- Il componente custom Daikin di HA va portato alla versione compatibile alla prossima occasione.
