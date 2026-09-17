---
title: "OCF Trainer U8: motivazioni AI grounded sul corpus normativo (corretta + esclusione delle errate)"
date: "2026-09-16"
author: "kos-domus"
status: "new"
tags: ["claude-api", "structured-outputs", "guardrails", "fastapi", "nextjs"]
session_type: "claude-code"
openclaw_version: ""
environment:
  os: "Linux (mini-PC)"
  ide: "Claude Code (VSCode extension)"
  model: "claude-fable-5-1"
---

## Objective
Rakki vuole associare lo studio alla pratica sui test: per ogni quesito OCF l'AI deve ricostruire dai testi normativi e dalla semantica della domanda perché la risposta ufficiale è corretta e perché ciascuna delle altre è esclusa.

## Steps Taken
### 1. Design (piano U8 riscritto)
Contesto = quesito + opzioni con la corretta marcata + articoli del corpus citati (comma in testa, max 6k caratteri). Structured output Pydantic via `client.messages.parse` (adaptive thinking, effort medium). Guardrail KTD-5: scrittura solo in `question_explanations(origin=ai)` + nuova `question_ai_meta`; citazioni ammesse solo tra gli articoli forniti (le altre scartate → `grounded=false` = "senza fonte nel corpus"); dissenso dall'ufficiale → `question_reports(ai_disagreement)`, risposta intoccabile; righe umane/verificate mai sovrascritte; idempotenza via `content_hash`.
**Result**: `app/ai/explain.py` + prompt `explanations_v1.md`, migrazione `b7e1c2d3f4a5`, CLI `ocf enrich` (limit/subject/ids/model/effort/dry-run/force/workers, costo per domanda e proiezione), endpoint `POST /questions/{id}/explanations/generate`, UI: chip citazioni ancorate al testo dell'articolo, badge "senza fonte", box "Da studiare", pulsante "Genera motivazioni con AI" in dettaglio e review, "segna come verificata".

### 2. Test
8 test unitari (client finto) + 1 API: happy, citazioni inventate scartate, dissenso → report senza toccare `is_correct`, rigenerazione che preserva umane/verificate (vincolo unico question/option/kind/origin → le verificate vincono), output non valido senza scritture parziali, normalizzazione articolo.
**Result**: 53 test verdi, tsc pulito.

### 3. Pilota reale (8 domande)
Sonnet 5: ~$0,015/domanda (~$75 per 4.997). Opus 5: ~$0,041/domanda (~$205); cita comma e lettera anche sui distrattori, Sonnet solo sulla corretta.
**Result**: gotcha risolto al primo giro: il modello scrive `article: "art. 1"` e la validazione esatta scartava tutto → `normalize_article()` prima del confronto.

### 4. U13 — editing + laboratorio prompt (stessa sessione, richiesta Rakki)
Decisione Rakki: Opus 5 sulle materie pesanti (B diritto mercato finanziario, A matematica/mercati), Sonnet 5 su C/D/E → run lanciato in background (`data/enrich-run-20260916.sh`, ~9 domande/min, ripristinabile: le già fatte vengono saltate per hash). Costruito: PATCH motivazione (→ `verified_by_user`, revisione), PATCH `key_concept` (`key_concept_by_user`, la rigenerazione lo conserva), tabella `ai_prompts` seedata da v1 con `GET/PUT /ai/prompts`, `POST /explanations/preview` (nessuna scrittura, costo restituito), pannello `PromptLab` nel dettaglio domanda (versione/modello/effort, editor, salva come versione/default, anteprima affiancata, adotta), `ocf enrich --prompt vX`. 57 test verdi.

## Lessons Learned
- Con le citazioni "solo dal contesto" la validazione deve normalizzare la forma dell'identificatore (art./articolo/spazi), altrimenti la fonte giusta viene buttata e il prodotto sembra "senza fonte".
- Il vincolo unico su `question_explanations` impone la regola "riga verificata vince sulla rigenerazione": non è solo una scelta di prodotto, è necessaria per non violare il vincolo.
- Il dev server API era un processo orfano senza `--reload`: le nuove route davano 404 finché non è stato riavviato.

## Files Touched
`backend/app/ai/{explain.py,prompts/explanations_v1.md}` · `backend/app/models/{core.py,__init__.py}` · `backend/alembic/versions/20260916_b7e1c2d3f4a5_ai_explanations_meta.py` · `backend/app/cli.py` · `backend/app/api/v1/{questions.py,sessions.py}` · `backend/tests/{test_ai_explain.py,test_api.py}` · `backend/pyproject.toml` (+anthropic) · `frontend/{lib/types.ts,lib/api.ts,components/QuestionReview.tsx,app/questions/[id]/page.tsx}` · `Makefile` (enrich, enrich-dry) · `plans/PLAN-ocf-trainer.md` · `DOSSIER.md`.

### 5. U14 — abbattimento costi e run completo (stessa giornata)
Il run Opus si è fermato a 310/4997 per credito API esaurito, dopo aver accumulato ~1.800 richieste rifiutate: aggiunto un guard che interrompe il batch su errori di credito/auth (exit 3). Poi misurazione con `count_tokens`: l'input fatturato (3.717 tok) era 965 system + 1.303 contenuto + **1.032 di schema JSON annidato** (31%). Tre interventi: schema snello a 479 tok con citazioni in formato `FONTE|ARTICOLO|COMMA`; A/B Opus vs Sonnet su 3 domande di diritto (stesso ragionamento e stesse fonti, comma meno preciso → chiuso irrigidendo il prompt); **Batch API** (`app/ai/batch.py`, −50%, stessi guardrail, stato in `data/ai-batches.json`).
**Result**: $155 → $28,69 stimati, $28,56 reali per 4.690 domande. Totale progetto $41,12 per 4.997/4.997 domande e 19.988 motivazioni. 2.129 domande con citazione dal corpus, 7 dissensi dalla risposta ufficiale aperti in Segnalazioni. 65 test verdi.

**Lezione aggiuntiva**: in una generazione di massa lo schema di structured output è un costo fisso moltiplicato per ogni richiesta — va misurato con `count_tokens`, non stimato. E un run lungo deve distinguere errori transitori da errori di account, altrimenti "fallisce con successo" per migliaia di richieste.

### 6. U15 — recupero fonti per le domande che non citano articoli + pagina Revisione
Richiesta Rakki: "motore di ricerca semantico". **Misurato invece di assunto**, con ground truth le 2.154 coppie domanda→articolo note: BM25 su SQLite FTS5 recall top-10 **73%**, embedding multilingua piccoli (fastembed MiniLM ONNX) **46%**, fusione RRF **71%**. Sul lessico giuridico italiano il lessicale batte il semantico e la fusione peggiora → `fastembed` disinstallato, zero dipendenze nuove, indice FTS5 in 0,2 s. Filtro per materia: nessun guadagno (72% vs 73%). Candidati 4 invece di 6: −25% input, −3 punti recall.

Architettura onesta: i candidati **non** diventano riferimenti. Entrano nel prompt in una sezione che dichiara che non sono citati nel quesito e possono essere fuori tema; il validatore accetta solo quelli forniti; se il modello ne cita davvero uno nasce un `question_references(origin='ai')` (badge "proposto da AI" già in UI), apribile e scartabile; uno scartato non torna fra i candidati. Prova reale su 6 domande: 4 ancorate con l'articolo esatto (art. 1936 c.c. fideiussione, art. 2409-bis c.c. revisione legale), 2 lasciate senza fonte perché nulla era pertinente.

**Pagina Revisione** (`/curation`): cinque code ordinate per resa (dissensi 7 · confidenza <0,5 107 · tue errate 30 · senza fonte 2.864 · mai confermate) con filtro materia e percentuale di avanzamento sulle confermate. Il filtro serve perché 1.495 delle "senza fonte" sono di matematica, dove la norma non c'entra.

**Result**: 70 test verdi. Batch B+D+E (931 domande, ~$5,20) inviati; resta C tributario (438, ~$4,83) in attesa di ricarica.

**Gotcha**: il dev server Next non registrava la rotta nuova (404 anche dopo attesa) → riavvio necessario; verificato con una GET reale, non solo controllando che il file esistesse.
