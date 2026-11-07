---
title: "Decision model locale (LLM + embedding) messo alla prova, pipeline SpesaBot sbloccata, Fase V con riferimento e mutanti"
date: "2026-11-06"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "performance", "testing", "debugging", "automation", "security", "cron"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Ubuntu 24.04 (mini PC, CPU AVX2, 32 GB RAM)"
  ide: "VS Code Remote SSH + Claude Code"
  model: "claude-opus-5-5"
---

## Objective

1. Capire se un modello di decisione locale (LLM piccolo su CPU con llama.cpp) e un modello di embedding locale possono filtrare i punti di una revisione cross-provider dei piani.
2. Sbloccare la pipeline settimanale di SpesaBot, uccisa due volte da systemd per timeout.
3. Preparare con il metodo «Fase V» (riferimento + mutanti + revisione di un altro provider) lo script di verifica di un piccolo servizio di download.

## Context

- Decisore: llama.cpp compilato dai sorgenti, server solo su 127.0.0.1, API a domande tipizzate (`noul`/`choice`). Set di valutazione: 331 punti di revisione etichettati, criteri G1–G6.
- Banchi precedenti: «riapre una decisione già presa» (G1) al livello del caso; «area di sicurezza» (G6) utile.

## Steps Taken

### 1. G1 come scelta multipla sul registro delle decisioni (LLM)
Domanda riformulata come scelta tra le voci del registro + «nessuna». Recupero BM25 troppo debole (voce giusta in 3/9) → registro intero (~58 opzioni).
**Result**: voce giusta 0/9, AUC 0,42; la probabilità si sparpaglia su troppe opzioni. ~263 s a punto.

### 2. Embedding locale per G1 (con gate di sicurezza)
Revisione CSO di tre modelli: due approvati con condizioni (GGUF ufficiali, sha256 fissati, solo loopback, `--no-slots`, servizio utente sandboxato), il terzo rinviato perché richiedeva una conversione con torch. Banco con `llama-server --embedding`.
**Result**: migliore AUC 0,65, voce giusta al primo posto 3/9. Conclusione: «riaprire una decisione» è contraddizione semantica, non somiglianza → resta al revisore umano/LLM grande.

### 3. Embedding per i doppioni (G5)
Per ogni punto, somiglianza con i punti precedenti dello stesso log (101 doppioni).
**Result**: come sì/no a soglia non basta (AUC 0,70); come **ricerca** funziona: l'originale è tra i primi 5 nel 91% dei casi, primo nel 61%.

### 4. Due stadi: embedding propone, LLM conferma la coppia
969 coppie (top-3), ~28 s a coppia su CPU.
**Result**: AUC 0,78; a precisione ~0,7 il richiamo raddoppia rispetto al solo embedding; primo posto giusto 57/74 contro 54/74. Ancora sotto la soglia d'uso (precisione 0,90): resta un aiuto che ordina i candidati.

### 5. Pipeline SpesaBot: diagnosi e correzione
- Log: dopo «Images healed (canonical)» il processo restava fermo fino al timeout (CPU quasi zero).
- Causa: `backfill_images_from_registry()` ripassava **tutti** i 13.760 SKU senza immagine accumulati, chiamando la lookup **due volte** per SKU (WHERE + SET); il ramo trigram usava `similarity() > 0.3`, che **non usa l'indice GiST** → ~0,4 s a chiamata → ~3 h.
- Fix 1 (db/026): lookup una sola volta per SKU (CTE), nuova funzione `_since(ts)` limitata agli SKU toccati dal giro (`updated_at >= inizio giro`), `statement_timeout 25min` via `PGOPTIONS`.
- Fix 2 (run-pipeline.sh): `trap 'trap - EXIT; record_finish timeout; exit 143' TERM INT`. Prima, dopo il SIGTERM di systemd, l'EXIT trap vedeva rc=0 (per i `|| echo non-fatal`) e registrava «success».
- Fix 3 (db/027): ramo trigram con `name_norm % x` (soglia impostata con `set_config('pg_trgm.similarity_threshold','0.3', true)`) + gli stessi filtri `similarity()`. Confronto su 420 SKU (anche con nomi alterati per forzare il ramo trigram): risultati identici.
**Result**: passo da >2 h a 13 min (fix 1) e poi a 1 min 50 s (fix 3).

### 6. Backup e buchi nei dati
- PostgreSQL non era nel backup restic (solo SQLite e cartelle): aggiunto un dump notturno dello schema dell'app prima del backup.
- Controllo giornaliero «ultimo giorno con prezzi nuovi per catena», con avvisi solo sui cambi di stato: ha trovato subito una catena ferma da 6 mesi e cinque mai raccolte.

### 7. Fase V di uno script di verifica
Servizio di download da un'API REST non ufficiale (login, elenco, «apri» e allegati via CDN). Script di verifica con tre server finti (API, CDN, origine estranea + proxy puntato all'estranea), riferimento scritto dal pianificatore, 44 varianti sbagliate.
**Result**: riferimento TUTTO OK, 44/44 varianti prese. La revisione cross-provider ha bocciato lo script due volte; dopo la seconda decide l'umano: corretti i difetti veri, il resto dichiarato fuori perimetro, esecuzione avviata con override esplicito registrato.

## Configuration Changes

- Nuovi modelli GGUF di embedding con sha256 in `SHA256SUMS`; banchi in `prove/` (embedding e coppie).
- SpesaBot: migrazioni db/026, db/027; `scripts/run-pipeline.sh` (inizio giro, timeout, trap TERM); commit locali.
- Timer utente: dump notturno del DB dell'app, controllo freschezza dei prezzi, pulizia oraria dei PDF vecchi nella cache documenti dell'agente di famiglia.
- Spento il desktop remoto GNOME (porta aggiuntiva ripartita dopo un riavvio); resta un solo server RDP.

## Key Discoveries

- **`similarity(a,b) > k` non usa l'indice trigram**; `a % b` sì. Con la soglia impostata a runtime i risultati non cambiano e la funzione diventa ~8× più veloce. `ALTER FUNCTION … SET pg_trgm.similarity_threshold` richiede superuser (parametro placeholder): `set_config(…, true)` dentro la funzione funziona.
- **`UPDATE … WHERE f(x) IS NOT NULL SET col = f(x)`** chiama la funzione due volte: meglio una CTE.
- **SIGTERM e trap EXIT in bash**: se il passo ucciso è seguito da `|| echo …`, lo script può uscire 0 e registrare un falso successo. Serve una trap esplicita su TERM.
- **`hermes send` copia sempre il messaggio inviato nella sessione di destinazione come messaggio dell'agente** (mirror senza opzione): testo non fidato dentro un invio entra nel contesto dell'agente.
- **Media policy dei gateway**: con `strict: false` un agente può allegare quasi ogni file della home fuori dalla denylist; l'allowlist scavalca la denylist, quindi va data per percorso esatto.
- Un LLM piccolo su CPU è utile per **classificare per area**, non per riconoscere contraddizioni; un embedding è utile per **cercare**, non per decidere da solo. In coppia migliorano, ma non bastano per filtrare in automatico ad alta precisione.
- Un'API che rifiuta gli allegati finché l'elemento non è stato «aperto» (404) costringe a una scelta esplicita sull'effetto collaterale «letto»: va decisa dall'utente e poi confinata (un solo percorso fisso, tetto per giro, mai ripetuto).

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| HTTP 500 su tutti i punti del banco LLM | input oltre il batch di 4.096 token | `-c 16384 -b 8192 -ub 8192`, stop dopo 3 errori |
| Round 2 della revisione non partito | il cap dei round conta anche quelli già nel log | `--rounds 2` |
| Una variante sfuggita al test | intestazioni controllate solo in uno scenario | controllo globale su tutte le richieste |
| Dump del DB fallito | lo schema di un'altra applicazione nello stesso DB non è leggibile dal ruolo | dump del solo schema dell'app (isolamento corretto) |

## Final State

- Decisore: G1 escluso; G5 = ricerca con embedding + conferma LLM come aiuto; prossimi banchi su smistamento posta e abbinamento prodotti tra catene.
- SpesaBot: pipeline corretta e più veloce, backup del DB, controllo dei buchi; prova vera al prossimo giro.
- Servizio di download: in esecuzione con il grafo (esecutore senza accesso ai test).

## Open Questions

- Abbinamento prodotti tra catene con embedding + LLM locali: supereranno la soglia del 98% di precisione?
- Copia dei backup fuori casa e pulsante unico di backup: da progettare con il CSO.
