---
title: "Excalidraw toolkit consolidation: design system, generator helpers, JSON spec DSL for cross-model diagram generation"
date: "2026-04-28"
author: "kos-domus"
status: "processed"
tags: ["excalidraw", "design-system", "dsl", "visual", "obsidian", "cross-model", "tooling"]
openclaw_version: "4.24"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "Claude Code (CLI + VSCode)"
  model: "claude-opus-4-7[1m]"
---

## Objective

Trasformare la generazione di diagrammi Excalidraw da pattern artigianale (un agente Claude scrive ~5000 token di JSON Excalidraw raw per ogni diagramma) a tooling deterministico riusabile da QUALSIASI modello (Codex CLI, Gemini, locale) a costo zero token Anthropic. Risolvere parallelamente tutte le sbavature visive accumulate in iterazioni precedenti (cilindro, ombre, label, frecce, palette).

## Context

- Vault Obsidian a `~/Obsidian-Personal/` è la knowledge layer cross-progetto, già attivo
- Esisteva un `excalidraw-generator.py` con helper Python per il design system canonico
- Esisteva un `system-architecture.excalidraw.md` come diagramma di reference, con sbavature visive da iterazioni precedenti (frecce mal centrate, label sovrapposte, palette mista yellow+lavender+peach)
- Vincolo costo: API Anthropic troppo cara per esporre il "diagram skill" agli agenti OpenClaw — serve un meccanismo cross-model deterministico
- Obiettivo durevole: la conoscenza visiva si accumula in `Decisions/excalidraw-design-system.md`, le primitive in `Templates/excalidraw-generator.py`, gli esempi in `Templates/diagrams/`

## Steps Taken

### 1. Palette coherence + 3D primitives

- Aggiunto `PALETTES` dict (lavender / sunny / mint / rose / sky), 4 ruoli per palette (primary / secondary / accent / shadow)
- Helper `palette('lavender')` per pescare token coerenti
- Aggiunte primitive 3D: `iso_cube(x, y, size, fill)` e `cylinder(x, y, w, h, fill)`
- Helper `_lighten()` / `_darken()` per derivare tinte dalle base color

### 2. Sticker shadow refinement

Ombra su `labeled_box(shadow=True)` rivista: stroke = saturated color (NON ink), `strokeWidth=2`, `roughness=0.7` (più pulito del front layer a 1.5). Risultato: l'ombra legge come "raised sticker" anziché blob piatto o seconda forma.

### 3. Fill styles + opacity zones + dark+white text

- Esposti `fill_style` (`solid` / `hachure` / `cross-hatch`) e `text_color` su `labeled_box`
- Aggiunto helper `zone_overlay(x, y, w, h, fill, opacity, label)` per regioni traslucide (cost zone, trust boundary, status region)
- Pattern "hero element": dark fill (`#5b21b6` deep purple from lavender family) + `text_color='#ffffff'`, max 1 per diagramma
- Showcase diagram `diagram-pipeline.excalidraw.md` mette in scena tutte le tecniche

### 4. JSON spec DSL — il punto chiave

- Scritto CLI `excalidraw-from-spec.py` che legge uno spec JSON (~30 righe) e produce il `.excalidraw.md` finale
- DSL primitive: `palette`, `legend`, `top_node`, `layers[]`, `side_nodes[]`, `arrows[]` con anchor `<id>` o `<layer_id>:layer[:edge]`
- Smart default per anchor: side_node → layer entra dal lato laterale (non dal centro-top, evita collisioni con primary flow)
- Routing modes: `direct`, `outside_left`, `outside_right` per polyline 5-waypoint
- Spec di riferimento: `Templates/diagrams/system-architecture.json` regenera identico al diagramma manuale (87 elementi, palette coerente, ombre, polyline corretti)
- Documentato in `Decisions/excalidraw-spec-dsl.md` per essere consumato da agenti non-Claude

### 5. Iterazione UI sui diagrammi (multipli round di feedback)

- Padding sublabel da +4 a +14 (visibile separazione dal box)
- Layer container LH 200 → 220 (più aria per sublabel)
- Legend default w da 240 → 290 + clearance 30px dal contenuto
- Hachure box: testo free-floating + pill bianco dietro (NON bound, altrimenti il pill maschera il testo)
- Polyline arrow con horizontal-first elbow per arrowhead pointing-down quando entra in spec.json's top edge
- Cilindro definitivo: ellisse piena darker in basso (NON solo arco-smile), ellisse lighter in alto, body neutro = effetto 3D battery

## Lessons Learned

### Architettura

1. **Separare ragionamento NL da generazione deterministica** è LA leva di costo per qualsiasi tool che genera artifact strutturati. ~17× token reduction vs raw JSON.
2. **DSL piccolo è meglio di DSL completo**: 5-7 primitive coprono l'80% dei diagrammi. Estendere quando un caso reale lo richiede, non a priori.
3. **CLI shell-callable + (opzionale) MCP wrapper**: il CLI funziona ovunque (qualsiasi agente con shell access), MCP è bonus per agenti MCP-aware.

### Design System

4. **One palette per diagramma**: token chromaticamente coerenti (primary / secondary / accent / shadow nella stessa hue family). Mai mescolare yellow + lavender + peach.
5. **Sticker shadow rule**: front=ink stroke su fill chiaro, shadow=stroke saturated color (NON ink), strokeWidth=2, roughness=0.7 (più pulito).
6. **Coerenza semantica via fillStyle**: nodi della stessa classe = stesso fillStyle. Distinzioni intra-classe vanno nel sublabel testuale.
7. **Max 1 hero per diagramma** (dark+white text). Più di uno = nessuno.
8. **Cilindro = ellisse piena darker in basso**, non smile arc. La "back arc" visibile dentro il corpo diventa volutamente l'effetto profondità.

### Layout / Anti-patterns

9. **Anti-collision frecce**: due primary flow che entrano allo stesso layer dal `top` center → arrowhead sovrapposti. Usa anchor espliciti `<id>:layer:left|right` per distribuire.
10. **side_node → layer**: smart default DEVE atterrare sul bordo laterale (right per side_node a destra), NON sul top center.
11. **Hachure + bound text incompatibili**: Excalidraw renderizza il testo bound col container, quindi pill drawn dopo maschera il testo. Soluzione: per pattern boxes, testo free-floating posizionato manualmente.
12. **label_bg opacity 100** (non 90): se vuoi mascherare la linea sotto, devi farlo davvero.
13. **Legend separation**: 30px di clearance orizzontale dal contenuto altrimenti si legge come parte del diagramma.

### Excalidraw rendering quirks

14. **Polyline con `roundness: {type: 2}`**: bezier smoothing, ma può overshoot vicino agli endpoint. Per cylinder silhouette è meglio polyline ad alta densità (N=30) senza smoothing.
15. **strokeWidth=0 + roughness=0** sul body del cylinder: necessario per coprire pulitamente l'arco posteriore dell'ellisse (anche con roughness 1.5 standard, microvariazioni lasciano "ghost line").
16. **Container binding != z-order indipendente**: un text bound a un rect viene renderizzato CON il rect, indipendentemente dalla sua posizione nell'array degli elements. Se serve un layer nel mezzo, bypassa il binding.

## Files Created / Changed

### Created
- `~/Obsidian-Personal/Templates/excalidraw-from-spec.py` — CLI deterministico (~280 righe)
- `~/Obsidian-Personal/Templates/diagrams/system-architecture.json` — spec di riferimento
- `~/Obsidian-Personal/Decisions/excalidraw-spec-dsl.md` — DSL reference per agenti
- `~/Obsidian-Personal/Decisions/diagram-pipeline.excalidraw.md` — showcase delle tecniche
- `~/Obsidian-Personal/Decisions/system-architecture-from-spec.excalidraw.md` — proof of equivalence

### Heavily updated
- `~/Obsidian-Personal/Templates/excalidraw-generator.py` — palette dict, 3D primitives, zone overlay, fill_style support, sticker shadow rule, cylinder rewrite
- `~/Obsidian-Personal/Decisions/excalidraw-design-system.md` — palette themes, sticker rules, fill styles, anti-patterns expansion
- `~/Obsidian-Personal/Decisions/system-architecture.excalidraw.md` — rigenerato

### Memory
- `~/.claude/projects/-home-kos-job-desk/memory/reference_excalidraw_design_system.md` — aggiornato con DSL CLI come modo raccomandato

## Open Follow-ups

- **Esporre fill_style / text_color / opacity / zones[] nel DSL JSON**: attualmente disponibili solo via Python helpers. Per agenti non-Claude serve aggiungerli allo spec schema.
- **MCP server wrapper** per `excalidraw-from-spec`: opzionale, ma renderebbe il tool disponibile a Cursor/Cline/Continue/altri agenti MCP-aware nativamente.
- **Altre palette themes** se servono casi specifici (monocromatico, alto contrasto per accessibilità).
- **Altri 3D primitives** se i diagrammi futuri richiedono pattern non coperti (server rack, cloud icon, document stack).
- **DSL extensions** quando emergono pattern: sequence diagrams, decision trees, mind maps. Il DSL v1 copre solo layered architecture.

## Key Insight per future sessioni

Il sistema ora regge una **knowledge base viva**: ogni nuovo pattern visivo che si scopre va in `excalidraw-design-system.md` come "regola" o "anti-pattern", ogni nuova primitive Python va in `excalidraw-generator.py`, ogni nuova primitive DSL va in `excalidraw-from-spec.py` + documentata in `excalidraw-spec-dsl.md` con un esempio in `Templates/diagrams/`. La conoscenza si accumula automaticamente — basta aggiornare i 4 file insieme quando si impara qualcosa.

Per regenerare un diagramma esistente dopo aggiornamenti al generator/CLI:
```bash
python3 ~/Obsidian-Personal/Templates/excalidraw-generator.py <topic>
# oppure (preferito per cross-model):
python3 ~/Obsidian-Personal/Templates/excalidraw-from-spec.py <spec.json> -o <name>
```
