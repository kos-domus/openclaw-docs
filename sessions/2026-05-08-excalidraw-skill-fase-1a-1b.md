---
title: "Excalidraw Diagrams Skill — Fase 1a kick-off + Fase 1b direction-horizontal + 7 iterazioni visive"
date: "2026-05-08"
author: "kos-domus"
status: "processed"
tags: ["skills", "claude-md", "automation", "configuration"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Linux 6.17.0-22-generic"
  ide: "VSCode"
  model: "claude-opus-4-7[1m]"
---

## Objective

Kick-off Fase 1a della skill `excalidraw-diagrams-skill` (DSL builder + corpus seed + smoke test) seguita da Fase 1b direction-horizontal feature, con iterazione visiva guidata da feedback diretto su 7 cicli (v1 → v7.1) per consolidare typography, spacing, pattern semantics, frecce routing.

## Context

Sessione precedente (mattina) aveva chiuso la design phase con pattern A specialist consultations + addendum decisioni autoritative D1-D7. Stato: scaffold creato, 4 esempi categoria A + 3 librerie MIT categoria B nel corpus, soft-block kick-off Fase 1 in attesa di decisione Rakki.

Driver use case: pubblicazione daily Instagram + skill distributable platform-agnostic.

## Steps Taken

### 1. Audit asset esistenti + scaffold check

Read della directory progetto + measurement CLI vault Obsidian (`excalidraw-from-spec.py` 365 lines, `excalidraw-generator.py` 1341 lines). Scaffold dirs già creati ma vuoti.

**Result**: ~1700 lines di codice DSL + helpers da refactorare in module importable.

### 2. Module refactor

Copiati 2 file CLI vault → `lib/helpers.py` + `lib/dsl.py`. Edit chirurgici:
- Rimosso import dinamico `importlib.util` → sostituito con `from .helpers import ...`
- Rimosso `main()` + entry point hardcoded path
- Aggiunto `write_excalidraw_standalone()` per output `.excalidraw` puro (NON Obsidian-wrapped)
- Aggiunto `CANVAS_PRESETS` dict (8 preset platform-agnostic)
- Creato `lib/__init__.py` con public API export

**Result**: Module importable. Smoke test 4/4 passed (import + inline spec build + vault spec regression + JSON shape + CLI subprocess).

### 3. CLI wrapper + render pipeline decision

Scritto `scripts/generate_excalidraw.py` come thin CLI wrapper su `lib.build()`. Per `scripts/export_scene.py` (rendering `.excalidraw → SVG → PNG`) decisione strategica: **stub Fase 1b/1c split** invece di forzare implementation.

**Razionale**: Excalidraw aesthetic hand-drawn dipende da roughjs (libreria JS). Un mini-renderer Python perderebbe l'identità sketchy (viola SOUL principio 1). Server-side faithful rendering richiede Node subprocess o Playwright headless — decisione tecnica matura merita spike dedicato, non rabbit hole nella stessa sessione.

**Result**: Output primario Fase 1a = `.excalidraw` standalone (semanticamente corretto, openable in Excalidraw web per export manuale SVG/PNG). Render automation deferred.

### 4. Template + style YAML (documentational)

`templates/layered_arch.yaml` + `styles/sketchy-handdrawn.yaml` come spec definitions + slot reference + best/avoid sections. NON runtime-applied in Fase 1a — Claude consulta come few-shot reference, l'utente popola lo spec direttamente.

### 5. Iterazione visiva v1 → v7.1 (7 cicli)

Pattern: l'utente apre il `.excalidraw` generato in Excalidraw web, vede sbavature, descrive in NL ("scritte troppo piccole", "sfondi monotoni", "caselle si toccano"), il code agent applica fix mirato in `lib/dsl.py` o `lib/helpers.py`, rigenera, l'utente ri-valida.

**Cicli**:

- **v1 baseline** — DSL generation funzionante, output formati corretti
- **v2 quick wins** — Typography +40% scale (44pt title, 22pt nodes, 16pt sublabels — vs vault default 32/18/12). Aggiunto `bg_pattern` canvas-wide come overlay rect.
- **v3 architectural fix** — Pattern bg-wide RIMOSSO (era invasivo), spostato come `layer.pattern` + `node.pattern` per-element. Semantica: hachure=draft, cross-hatch=deprecated, dots=external dependency. Spacing bumped LH 220→290, GAP 70→90.
- **v4 centratura unified** — `labeled_box` ora usa SEMPRE free-floating text + manual centering (eliminato bound text Excalidraw che era inaffidabile a typography +40%). Spacing ulteriore: LH→360, GAP→120, LAYER_X/W ricalibrati per evitare overlap legend in canvas 1320.
- **v5 BUG CRITICO**: `_resolve_anchor` + outside routing avevano hardcoded magic numbers (140 = old LAYER_X, 1080/1100 = old LAYER_RIGHT). Quando in v4 ho cambiato i constanti, le funzioni di arrow routing non si sono aggiornate → frecce edge-of-layer puntavano fuori dai bordi reali. Fix: sostituiti con LAYER_X/LAYER_RIGHT dinamici. Plus pattern opacity 15→8 per equity visiva tra layer.
- **v6 direction horizontal** — Implementata feature `direction: vertical | horizontal` nel DSL. Layer container in horizontal stack LEFT→RIGHT, nodi distribuiti TOP→BOTTOM. HORIZONTAL constants (HLAYER_FIRST_X, HLW=380, HLH=600, HGAP=120). `grid_y_positions` helper. Branch in `build()` per layer iteration + node positioning. `_build_arrow(direction)` con default sides direction-aware (vertical bottom→top, horizontal right→left). Side_nodes: in horizontal `side='right'` = below layer, `side='left'` = above. Back-compat 100% — vertical regression test passa byte-identical.
- **v7 polish** — 3 fix: layer pattern split fill (opacity 8) + border (opacity 70) per equity visiva. HLH 760→600 per side_node room. Smart-side selection direction-aware per side_node→layer arrow routing in horizontal mode.
- **v7.1** — `HSIDE_GAP 50→100` per dare aria al label arrow ("metrics" su Grafana→TRANSFORM era schiacciato sulle box).

### 6. Palette audit

Generati 7 file `outputs/palette-demo/<palette>.excalidraw` — uno per ogni palette del design system (lavender default, sunny, mint, rose, sky, sunset-dark, spesify-brand) usando lo stesso spec base. `sunset-dark` con `bg_color=#0a0a0a` (canvas dark).

**Result**: stress test multi-palette per identificare drift su palette non-default.

## Configuration Changes

- Nuovo module `lib/{__init__,dsl,helpers}.py` (DSL builder + low-level primitives)
- Nuovo `scripts/generate_excalidraw.py` (CLI wrapper)
- Nuovo `scripts/export_scene.py` (stub Fase 1b)
- Nuovi `templates/layered_arch.yaml` + `templates/horizontal_flow.yaml`
- Nuovo `styles/sketchy-handdrawn.yaml`
- Nuovo `requirements.txt` (PyYAML only Fase 1a)
- Nuovo corpus seed in `examples/`: 4 esempi categoria A + 3 librerie MIT categoria B
- 7 file palette demo in `outputs/palette-demo/`
- Aggiornato `lib/dsl.py` con HORIZONTAL constants + direction branch
- Aggiornato `lib/helpers.py` con `grid_y_positions` + layer_container split fill/border

## Key Discoveries

- **Iterazione visiva è feature, non bug**: 7 cicli di feedback NL → fix code → ri-render sono il workflow naturale per skill che produce artifact visivi. Documenta il pattern come "expected" per progetti simili — il patch engine Claude-mediated previsto Fase 1b automatizzerà esattamente questo loop.
- **Bound text Excalidraw inaffidabile a typography custom**: il bound text con `verticalAlign: middle` non centra correttamente quando text size diverso da container default. Fix robusto: free-floating text + manual centering matematico (`text_y = y + (h - text_h) / 2`). Sempre.
- **Magic numbers sparsi = bug futuro garantito**: il bug v5 nasce da hardcoded 140/1080 in `_resolve_anchor` + `_build_arrow` non aggiornati quando i layer constants sono cambiati. Lesson: tutti i constanti spaziali centralizzati in cima al module, mai duplicati.
- **Layer pattern equity visiva richiede split**: rect unico con `opacity=8` rende invisibile anche lo stroke dashed. Soluzione: 2 rect separati (fill pattern low-opacity + border high-opacity).
- **Smart-side selection direction-aware**: side_nodes hanno semantica diversa per direction (`right` = right-of-column in vertical, below-row in horizontal). Arrow routing deve adattarsi.
- **Roughjs hand-drawn aesthetic non replicabile in Python**: confermato che render server-side richiede ecosystem JS (Node/Playwright). Mini-renderer Python perderebbe l'identità del progetto. Decisione architetturale di Fase 1b spike Node vs Playwright.
- **lzstring decompression per Obsidian Excalidraw plugin**: il format `.excalidraw.md` Obsidian usa lz-string compression (NON zlib). Per estrarre `.excalidraw` standalone serve `pip install lzstring` + `decompressFromBase64()` + sanitize lone surrogates UTF-16 artifacts.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `zlib.error: Error -3 incorrect header check` su decompress Obsidian `.excalidraw.md` | Format è lz-string base64, non zlib | `pip install lzstring`; `LZString().decompressFromBase64(blob)` |
| `UnicodeEncodeError: surrogates not allowed` su scrittura JSON post-decompress | lz-string decode produce UTF-16 surrogates lone | `s.encode('utf-8', errors='ignore').decode('utf-8')` per sanitize |
| Frecce edge-of-layer puntano fuori dai bordi reali in v4 | Magic numbers 140/1080 hardcoded in `_resolve_anchor` non aggiornati post-LAYER_X bump | Sostituiti con LAYER_X/LAYER_RIGHT dinamici, + L['left']/L['right'] dal layer registry |
| Side_node Grafana fuori canvas in horizontal | HLH 760 + side_gap 50 + h 120 = 930 > canvas 1080 | HLH→600, HSIDE_GAP→100 (poi v7.1 100 per breath label arrow) |
| Bordo dashed layer pattern invisibile | Single rect con opacity 8 + dashed stroke → stroke ottiene anche opacity 8 | Split in 2 rect: fill (pattern, opacity 8, transparent stroke) + border (transparent fill, dashed stroke, opacity 70) |
| Freccia "metrics" mal-routed in horizontal | Smart-side selection hardcoded per vertical (right→left, left→right) | Branch `direction == 'horizontal'`: side='right' (= below) → src `top`, dst `bottom` |

## Final State

**Skill `excalidraw-diagrams-skill` v7.1**:
- DSL importable + CLI wrapper + corpus seed (4 originals + 3 MIT libraries)
- 2 modalità: `direction: vertical` (default) + `direction: horizontal`
- 8 canvas presets platform-agnostic
- Pattern semantic per layer/node/side_node (hachure/cross-hatch/dots/zigzag/...)
- Typography +40% scale per social readability
- Smart-side arrow routing direction-aware
- 7 palette colore (lavender/sunny/mint/rose/sky/sunset-dark/spesify-brand)
- Output `.excalidraw` standalone openable in Excalidraw web

**Smoke test passing**:
- vertical regression byte-identical post v6 refactor
- horizontal CI/CD 4-layer + horizontal data pipeline 3-layer + side_node + pattern
- 7 palette demo files

**Render pipeline (`.excalidraw → SVG → PNG`)**: deferred Fase 1b — output primario è `.excalidraw` openable in Excalidraw web per export manuale.

## Open Questions

- Render pipeline: Node subprocess (`@excalidraw/utils.exportToSvg`) vs Playwright headless? Spike pending (~30 min ognuno per decidere).
- Direction `mixed` (matrix layout layer×row, swim lane / org chart): feature non-trivial (~2h). Priority?
- Logo embed via Excalidraw native libraries: DSL extension `node.logo: "postgres" | ...` + `resolve_logo.py` lookup contro `examples/icon-libraries/`. ~1.5h.
- Patch engine `revise.py` Claude-mediated: NL feedback → JSON Patch → re-export. Automatizza il loop iterativo manuale fatto in questa sessione (7 cicli). ~1.5-2h.
- Real-time editing `.excalidraw` in IDE con Claude che vede modifiche: feature richiesta dall'utente. Fattibilità: alta via file watcher + diff intelligence, ma richiede MCP server dedicato o pattern manuale "modifica + saluta Claude". Da progettare.
