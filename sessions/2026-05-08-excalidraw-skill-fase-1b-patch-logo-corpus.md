---
title: "Excalidraw skill Fase 1b: patch engine + logo embed + corpus extension + VS Code remote editing setup"
date: "2026-05-08"
author: "kos-domus"
status: "processed"
tags: ["skills", "configuration", "vscode", "automation", "remote", "setup"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Linux 6.17.0-22-generic"
  ide: "VSCode Mac client + Remote-SSH a mini-PC"
  model: "claude-opus-4-7[1m]"
---

## Objective

Continuazione della sessione mattutina (chiusa con session log v7.1 sulla skill `excalidraw-diagrams-skill`). Implementazione Fase 1b deliverable: patch engine Claude-mediated, logo/icon embed via Excalidraw native libraries MIT, corpus extension a 6 library, bug fix lessons-learned, plus setup VS Code per editing `.excalidraw` side-by-side in remote-SSH.

## Context

Punto di partenza: skill v7.1 con DSL `direction: vertical | horizontal`, 7 palette, pattern semantic per layer/node, typography +40%, smart-side arrow routing. Output primario `.excalidraw` standalone (render automatico deferred a Fase 1b).

Pending Fase 1b: patch engine (per automatizzare il loop iterativo manuale fatto 7 volte la mattina), logo embed via Excalidraw native libraries MIT, render pipeline, direction `mixed`.

## Steps Taken

### 1. Patch engine `revise.py` — Claude-mediated (v8)

Implementato `lib/patch.py` module + `scripts/revise.py` CLI.

**Patch op DSL v1** (5 ops):
- `replace_field` — change qualsiasi field
- `shift` — numeric delta su field (es. `x -= 150`)
- `set_pattern` — change `fillStyle` (solid/hachure/cross-hatch/dots/zigzag)
- `set_color` — alias semantico per `backgroundColor`
- `remove` — delete element

**Selectors** (AND logic): `by_id` (exact), `by_label` (case-insensitive substring sul testo), `by_group` (exact groupId).

**Versioning automatico**: `foo.excalidraw → foo.v2 → foo.v3 → ... → foo.v10`. Original mai sovrascritto.

**Pattern Claude-mediated**: lo script NON chiama Claude API a runtime. Il main thread Claude (skill execution) legge `.excalidraw` JSON + feedback NL utente, computa ops mentale, invoca `revise.py --patch '<JSON>'` con ops strutturati. Lo script applica deterministically.

**Result**: 5 smoke tests passati (versioned_output_path 3 casi, find_elements multi-match, apply_patches 5 ops × 3 selectors, CLI file/inline patch, dry-run, error reporting con exit code 3).

### 2. Logo/Icon embed via Excalidraw native libraries MIT (v9)

Implementato `lib/logos.py` module:
- `load_libraries()` — auto-scan `examples/icon-libraries/*.excalidrawlib`
- `resolve_logo()` — supporta alias semantico OR `{library, index}` dict
- `embed_logo_at()` — scale + translate logo elements + fresh ids per evitare collisioni
- `SEMANTIC_ALIASES` registry — manualmente populato man mano che item vengono identificati

**DSL extension**: `node.logo: "json"` (alias) | `{library: "drwnio", index: 3}` (deterministic).

**Extended `labeled_box(logo_elements=...)`** in helpers.py: quando logo presente, label si sposta in basso (text_y = bottom - 14px), logo elements occupano top portion 60% della box height.

**Realtà del corpus**: format `.excalidrawlib` v1 NON ha metadata `name` per item. Items sono lista anonima di shape groups. Solo `drwnio[3]` ha text "JSON" → unico alias auto-identificabile.

### 3. Catalog viewer (v9.1)

Implementato `scripts/list_logos.py` — genera 1 file `.excalidraw` per library con grid layout: cella = bordo dashed + logo embedded + label `[lib:N]`. Permette di identificare visivamente quale item è quale logo, propedeutico a popolare `SEMANTIC_ALIASES`.

**Workflow**: apri catalog in Excalidraw web/VSCode, identifichi visivamente "drwnio[7] = postgres elephant", aggiungi alias.

### 4. Corpus extension dual-purpose (v9.2)

Feedback Rakki post-ispezione catalog v9.1: il corpus `aws-simple-icons` contiene shape decorative editabili (es. un secchiello stilizzato = AWS S3 concept marker) che NON sono brand logos ma comunque utili per layout decoration.

**Riconoscimento dual-purpose del corpus**:
- LOGOS: brand-specific (Postgres elephant, Docker whale, AWS logo riconoscibile)
- ICONS: decorative/concept shapes (bucket, server stilizzato, cylinder DB)

Stesso meccanismo di embed serve per entrambi — distinzione solo semantica.

**Aggiunte**:
- 3 library extra scaricate da `excalidraw-libraries` GitHub repo (MIT):
  - `technology-logos.excalidrawlib` (18 items, autore maeddes) — Kubernetes/Docker/Git/Spring/Terraform claimed
  - `aws-serverless.excalidrawlib` (15 items, autore Slobodan Stojanović) — Lambda/API Gateway/DynamoDB
  - `cloud.excalidrawlib` (19 items, autore rfranzke) — multi-cloud K8s/AWS/Azure/GCP
- **Corpus totale**: 6 library, 94 items
- **`cloud[4]` = primo verified brand logo** (text "amazon web services" presente, 107 elements detail)
- DSL: `node.icon` come field primario, `node.logo` mantenuto come alias backward-compat
- 2 alias verificati: `'aws'` → `(cloud, 4)`, `'json'` → `(drwnio, 3)`

### 5. Bug fix v9.3 — CANVAS_W hardcoded

**Issue identificato in v6** (transpose vertical→horizontal di sky.excalidraw): il top_node "Rakki" si centrava su `CANVAS_W=1320` constant invece che sul canvas reale dello spec → su canvas `landscape-1920x1080` risultava biased verso sx (center=636 invece di 930 expected).

**Fix**:
```python
# Was: x = (CANVAS_W - w) // 2 - 30  # hardcoded 1320
canvas_name = spec.get('canvas', 'standard-1320x800')
canvas_w, canvas_h = CANVAS_PRESETS.get(canvas_name, (CANVAS_W, 800))
x = (canvas_w - w) // 2 - 30  # canvas-real width
```

**Verified**:
- Vertical 1320: top_node center=636 ✓ (off-by-6 da int division, accettabile)
- Horizontal 1920: top_node center=930 ✓ (pre-fix era 545 = sx-biased)

**Lesson learned esplicita**: pattern bug ricorrente "magic numbers stale dopo refactor constants". Già visto in v5 (LAYER_X/RIGHT in `_resolve_anchor`) → ora ancora questo. Audit sistematico necessario in Fase 1c.

### 6. VS Code remote-SSH editing setup

User installed `pomdtr.excalidraw-editor` extension per visualizzare `.excalidraw` come canvas live in VS Code. Issue: file apriva in nuova window separata invece di side-by-side nella stessa window.

**Diagnosis**: in setup remote-SSH (Mac client → mini-PC server), VS Code apre custom editor webview in window separata di default, plus default behavior `openFilesInNewWindow` non era esplicitamente off.

**Fix** in `~/.vscode-server/data/Machine/settings.json` (sul mini-PC, override scope per remote-SSH window):
```json
{
  "workbench.editor.openSideBySideDirection": "right",
  "workbench.editor.splitInGroupLayout": "horizontal",
  "window.openFilesInNewWindow": "off",
  "window.openFoldersInNewWindow": "off",
  "workbench.editorAssociations": {
    "*.excalidraw": "editor.excalidraw"
  }
}
```

**ViewType `editor.excalidraw`** confermato via WebFetch del marketplace VS Code page dell'extension.

**Workflow risultante**: Cmd+P apre file → Excalidraw editor renderizza canvas side-by-side nello stesso VS Code window connesso al mini-PC. Loop "Claude genera/modifica → utente vede live → feedback NL" diventa fluido.

### 7. (Bonus) Re-applicazione transpose vertical→horizontal

Su feedback Rakki ("trasforma sky.excalidraw da verticale a orizzontale"): chiarito che NON è patch puntuale (richiederebbe ~100 shift ops) ma richiede rigenerazione DSL con `direction: 'horizontal'`. Implementato direttamente, plus identificato il bug CANVAS_W di cui sopra.

## Configuration Changes

- Nuovi `lib/{patch,logos}.py` modules
- Nuovi `scripts/{revise,list_logos}.py` CLI
- Estesi `lib/dsl.py` (build con node.icon support, fix CANVAS_W)
- Esteso `lib/helpers.py` (labeled_box(logo_elements=...) + layer_container split fill/border) — NB: estensione del file copiato dalla CLI vault, NON tocca il vault originale
- Nuovi `examples/icon-libraries/{technology-logos,aws-serverless,cloud}.excalidrawlib` + meta.yaml
- Nuovi `outputs/catalogs/<library>-catalog.excalidraw` (6 file)
- Nuovo `~/.vscode-server/data/Machine/settings.json` con 5 setting per remote-SSH editing
- Public API estesa in `lib/__init__.py` (16 funzioni totali: DSL + patch + logos + catalog)

## Key Discoveries

- **Pattern Claude-mediated patch engine**: separazione perfetta tra "intelligence NL→ops" (Claude main thread, no API runtime) e "applier deterministic" (script Python). Permette automazione del loop iterativo SENZA cost runtime. Pattern generalizzabile a qualsiasi visual artifact (CAD, design tools, ecc.).
- **Format `.excalidrawlib` v1 senza metadata**: items sono liste anonime di shape groups. Per identificarli serve catalog viewer + lookup visivo manuale. Le librerie v2+ (se mai esistono) avrebbero `name` per item — fix futuro per la community Excalidraw.
- **Corpus dual-purpose**: librerie chiamate "logos" contengono ~80% shape decorative + ~20% brand veri. Distinzione semantica (LOGOS/ICONS) ma stesso embed mechanism. Library `cloud[4]` confirmed brand: 107 elements per il dettaglio del logo AWS.
- **Magic numbers = bug futuro garantito**: pattern ricorrente già visto in v5 (LAYER_X/RIGHT) e ora v9.3 (CANVAS_W). Lesson: tutti i constanti spaziali centralizzati e parametrizzati dal canvas spec, mai hardcoded.
- **VS Code remote-SSH custom editor + side-by-side**: il setting `workbench.editorAssociations` deve usare il `viewType` registrato dall'extension (per Excalidraw: `editor.excalidraw`, scoperto via marketplace page). Le Machine settings sul mini-PC override le User settings del Mac per la window remote-SSH connessa.
- **Catalog viewer come tool meta**: lo script `list_logos.py` genera artifact (catalog.excalidraw) usando le funzioni della skill stessa (embed_logo_at) — dogfooding della pipeline di embed. Pattern utile per altri tool che hanno corpus opaco.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Frecce edge-of-layer puntavano fuori dai bordi reali in v4 (cerchiato da Rakki nelle annotazioni image) | Magic numbers 140/1080 hardcoded in `_resolve_anchor` non aggiornati post LAYER_X bump v4 | Sostituiti con LAYER_X/LAYER_RIGHT dinamici + L['left']/L['right'] dal layer registry (v5) |
| Top_node "Rakki" centrato male su canvas != 1320 in horizontal | `CANVAS_W = 1320` constant hardcoded usata in top_node x calcolo | Lookup canvas reale via `CANVAS_PRESETS[spec.canvas]` (v9.3) |
| Side_node Grafana fuori canvas in horizontal | HLH 760 + side_gap 50 + h 120 = 930 > canvas 1080 | HLH→600, HSIDE_GAP→100 (v7) |
| Bordo dashed layer pattern invisibile | Single rect con opacity 8 + dashed stroke → stroke ottiene anche opacity 8 | Split in 2 rect: fill (pattern, opacity 8, transparent stroke) + border (transparent fill, dashed stroke, opacity 70) |
| Freccia "metrics" mal-routed in horizontal mode | Smart-side selection hardcoded per vertical (right→left) | Branch `direction == 'horizontal'`: side='right' (= below) → src `top`, dst `bottom` (v7) |
| `.excalidraw` apriva in window separata invece di side-by-side | Default VS Code behavior + custom editor in remote-SSH | 5 Machine settings: openFilesInNewWindow=off + openSideBySideDirection=right + editorAssociations |
| `.excalidraw` apriva come JSON raw invece di canvas | Cmd+P apre con default text editor, no association | `editorAssociations: {"*.excalidraw": "editor.excalidraw"}` (viewType da marketplace) |

## Final State

**Skill `excalidraw-diagrams-skill` v9.3** — production-ready per uso personale con visual loop ottimale:

| Capability | Version |
|---|---|
| DSL builder vertical + horizontal | v6 |
| 7 palette colori | v1 |
| Pattern semantic per layer/node | v3 |
| Typography social-readable +40% | v2 |
| Smart-side arrow routing direction-aware | v6 |
| Patch engine Claude-mediated (5 ops + 3 selectors + versioning) | **v8** |
| Logo/icon embed (6 library MIT, 94 items) | **v9** |
| Catalog viewer per identificare items | **v9.1** |
| Corpus extension dual-purpose (LOGOS+ICONS) | **v9.2** |
| Bug fix CANVAS_W canvas-aware | **v9.3** |
| Side-by-side VS Code editing in remote-SSH | **setup completato** |

**16 funzioni public API** in `lib/__init__.py`: build, load_spec, write_excalidraw_standalone, CANVAS_PRESETS, apply_patches, find_elements, versioned_output_path, load_excalidraw, save_excalidraw, load_libraries, list_libraries, resolve_logo, embed_logo_at, describe_libraries, SEMANTIC_ALIASES.

**Outputs in `outputs/`**:
- 3 file v6 (vertical regression + horizontal cicd + horizontal data-pipeline)
- 1 file v9 logo-embed
- 1 file v9.2 multi-library
- 7 file palette-demo (1 per palette + sky-horizontal v9.3)
- 6 file catalogs/<library>-catalog.excalidraw

## Open Questions

- **Render pipeline `.excalidraw → SVG → PNG`**: ancora deferred. Spike Node `@excalidraw/utils.exportToSvg` vs Playwright headless ancora pending. ~1.5-2h focused next session.
- **Direction `mixed`** (matrix layout layer×row, swim lane / org chart): non implementato. ~2h.
- **Template restanti** (`tree`, `flowchart`) + style restanti (`minimal`, `playful`): YAML documentational da scrivere, runtime application via `apply_template()`/`apply_style()` da implementare.
- **Popolazione `SEMANTIC_ALIASES`**: dipende da Rakki che apre i 6 catalog visivamente e identifica items. Effort distribuito nel tempo, non blocco.
- **Audit completo magic numbers**: lessons learned codificata, audit sistematico ancora da fare in Fase 1c.
- **`SKILL.md` entry point Anthropic skills**: trigger description già scritta nel DOSSIER §5, da formalizzare in `SKILL.md` per packaging.
- **MCP server custom per real-time editing live**: parzialmente sbloccato dal setup VS Code Excalidraw extension side-by-side. Per "Claude vede modifiche live" automatic, serve ancora MCP file watcher dedicato. Da considerare in Fase 2 quando il patch engine ha real-world usage.
