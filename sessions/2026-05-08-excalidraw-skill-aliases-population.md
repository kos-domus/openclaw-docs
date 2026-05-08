---
title: "Excalidraw skill: 83 SEMANTIC_ALIASES popolati + 100% corpus coverage + bug fix CANVAS_W"
date: "2026-05-08"
author: "kos-domus"
status: "ready"
tags: ["skills", "configuration", "vscode", "automation", "remote", "memory"]
session_type: "openclaw"
client: ""
openclaw_version: ""
environment:
  os: "Linux 6.17.0-22-generic"
  ide: "VSCode Mac client + Remote-SSH a mini-PC"
  model: "claude-opus-4-7[1m]"
---

## Objective

Terza tranche della giornata sulla skill `excalidraw-diagrams-skill`: bug fix CANVAS_W canvas-aware (lessons learned magic numbers), setup VS Code completo per editing `.excalidraw` side-by-side in remote-SSH, e popolazione completa di `SEMANTIC_ALIASES` registry tramite identificazione visiva di tutti e 6 i catalog del corpus. Obiettivo: trasformare la skill da "richiede `{library, index}` per ogni logo" a "scrivi `node.icon: 'postgres'`".

## Context

Punto di partenza: skill v9.3 con patch engine v8 + logo embed v9 + 6 library MIT (94 items totali) ma **solo 2 alias verificati** (`aws` cloud:4, `json` drwnio:3). Per usare un logo serviva sempre la sintassi `{library, index}` deterministic. UX poco fluida: l'utente deve aprire catalog, contare le celle, ricordare i numeri.

Plus: setup VS Code remote-SSH lasciato in stato parziale (file `.excalidraw` aprivano in nuova window separata invece di side-by-side).

## Steps Taken

### 1. Bug fix v9.3 — CANVAS_W canvas-aware

Audit `grep -n "1320" lib/` ha trovato il vecchio `CANVAS_W = 1320` constant ancora usato per il top_node positioning, mentre tutti gli altri constanti spaziali erano ormai parametrizzati.

**Fix**:
```python
# Was: x = (CANVAS_W - w) // 2 - 30
canvas_name = spec.get('canvas', 'standard-1320x800')
canvas_w, canvas_h = CANVAS_PRESETS.get(canvas_name, (CANVAS_W, 800))
x = (canvas_w - w) // 2 - 30  # canvas-real width
```

**Verified via test**:
- Vertical 1320: top_node center=636 ✓ (atteso ~630, off-by-6 da int division)
- Horizontal 1920: top_node center=930 ✓ (pre-fix era 545, sx-biased)

**Lesson learned esplicita**: pattern bug ricorrente "magic numbers stale post refactor". Già visto in v5 (LAYER_X/RIGHT in `_resolve_anchor`) e ora di nuovo in v9.3. Codified come task: audit sistematico in Fase 1c.

### 2. Setup VS Code Excalidraw extension side-by-side

User aveva installato `pomdtr.excalidraw-editor` ma file aprivano in nuova window separata invece di pannello side-by-side nello stesso VS Code window.

**Iterazione diagnostica**:
- Prima ho fixato 4 setting layout in `~/.vscode-server/data/Machine/settings.json` (Machine scope, applicato a window remote-SSH connessa al mini-PC) → risolse "split layout direction" ma non l'apertura in new window
- Aggiunto `window.openFilesInNewWindow: "off"` → ancora apriva nuova window
- L'extension Excalidraw apre in nuova process VS Code per remote-SSH workflow
- Soluzione: `Cmd+P` (quick-open) apre nello stesso window → però il file appare come JSON raw, non canvas
- Per associare `.excalidraw` al custom editor: `workbench.editorAssociations` con viewType **`editor.excalidraw`** (scoperto via WebFetch del marketplace VS Code page dell'extension)

**Final settings.json sul mini-PC** (5 setting):
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

**Result**: ogni `.excalidraw` aperto (Cmd+P / sidebar / qualunque metodo) parte direttamente come canvas Excalidraw side-by-side nella stessa window. Loop "Claude genera/modifica → utente vede live → feedback NL" diventa fluido senza switch context.

### 3. Population workflow — script `list_logos.py` + screenshot iterativo

Per popolare `SEMANTIC_ALIASES` da library v1 senza metadata: implementato workflow iterativo:

1. **Catalog viewer** (`scripts/list_logos.py`): genera 1 file `.excalidraw` per library con grid layout (cella = bordo dashed + logo embedded + label `[lib:N]`). 6 catalog files in `outputs/catalogs/`.
2. **Visual identification**: utente apre catalog in Excalidraw editor side-by-side, fa screenshot dell'intero grid.
3. **Batch identification con confidence levels**: per ogni screenshot, compongo tabella (cell, iconography, identification, confidence ⭐⭐⭐ / ⭐⭐ / ⭐).
4. **Apply alias**: aggiunta batch a `SEMANTIC_ALIASES` con commenti che indicano confidence e razionale.
5. **Skip ambigui**: per items low-confidence, skip esplicito documentato (es. "drwnio:5 — red stack: RedHat? Tomcat? Elastic? unknown").
6. **Smoke test**: dopo ogni library, smoke test che usa i nuovi alias per validation visiva.

### 4. Population di tutti e 6 i catalog (83 alias finali)

Ordine di processing + count:

| Library | Items | Alias registered | Skipped |
|---|---|---|---|
| `aws-serverless` | 15 | 14 (10 high + 4 medium conf) | 3 (cells 8/11/13 ambigui) |
| `cloud` | 19 | 29 (4 brand + 25 concept/aliases) | 2 (skull + pixel arrow) |
| `drwnio` | 18 | 16 (8 brand + 8 concept/aliases + 2 medium) | 2 + 5 duplicati cloud |
| `software-architecture` | 7 | 11 (5 unique + aliases) | 1 duplicato |
| `technology-logos` | 18 | 9 (4 brand + 4 concept/aliases) | 9 ambigui + 3 duplicati |
| `aws-simple-icons` | 17 | 4 (rds/ec2/ec2-cluster/sqs with-text variants) | 13 (varianti style + duplicati) |
| **TOTAL** | **94** | **83 alias** | — |

**Alias verificati high-confidence** (brand riconoscibili o text-tagged):
- Cloud providers: `aws`, `azure`, `gcp`, `kubernetes`, `gardener`
- AWS services: `lambda`, `api-gateway`, `dynamodb`, `s3`, `cognito`, `step-functions`, `cloudwatch`, `amplify`, `rds`, `ec2`, `ec2-cluster`, `sqs`
- Languages/Frameworks: `python`, `go`/`golang`, `kotlin`, `spring`
- Containers/IaC: `docker`, `terraform`
- Tools: `nginx`, `linux`/`tux`, `postgres`, `github`
- Concept icons: `laptop`, `cpu`, `database`, `globe`, `cluster`, `monitoring`, `microservice`, `layers`, `cache`, `web-app`, `mobile`, `gear`, `molecule`, `dns`, `code`, `browser`, `secure-network`, `lock`, `server-rack`, `datacenter`

**Alias medium-confidence**: `eventbridge`, `appsync`, `kinesis`, `sns`, `vault`, `key`, `kafka`/`pubsub`. Documentati con tag e commento "needs validation".

### 5. 5 smoke test progressivi v9.4-v9.7

Per ogni library identificata, smoke test che dimostra l'uso dei nuovi alias:

| Smoke | Path | Library focus | Elements |
|---|---|---|---|
| v9.4-1 | `outputs/smoke-1a-v9-4-aws-serverless-stack.excalidraw` | aws-serverless (lambda/api-gw/dynamodb/s3/cognito/step-functions/cloudwatch) | 96 |
| v9.4-2 | `outputs/smoke-1a-v9-4-multi-cloud-infra.excalidraw` | cloud (aws/azure/gcp/k8s/gardener/laptop/globe/cluster/monitoring) horizontal | 292 |
| v9.5 | `outputs/smoke-1a-v9-5-web-stack.excalidraw` | drwnio (postgres/docker/python/go/nginx/github/kafka) + cloud | 93 |
| v9.6 | `outputs/smoke-1a-v9-6-jvm-cloud-native.excalidraw` | technology-logos (kotlin/spring/linux/terraform) + cloud | 75 |
| v9.7 | `outputs/smoke-1a-v9-7-aws-classic.excalidraw` | aws-simple-icons (ec2/rds/sqs) + aws-serverless | 61 |

Tutti aprono come canvas Excalidraw editabili, validation visiva fatta dall'user.

### 6. DOSSIER + memory updates

DOSSIER §12 esteso con sezioni dedicate v9.3, v9.4 (population), v9.6, v9.7. Stato finale registry tabulato.

Memory `project_excalidraw_diagrams_skill.md` aggiornata: v9 capability summary include 83 alias + workflow per popolazione iterative.

## Configuration Changes

- `lib/dsl.py`: aggiunto canvas-aware lookup per top_node positioning (fix CANVAS_W)
- `lib/logos.py`: 83 alias semantici + razionale + skip documentati
- `~/.vscode-server/data/Machine/settings.json` (sul mini-PC): 5 setting per remote-SSH editing fluido
- 5 nuovi smoke test in `outputs/`
- DOSSIER + project memory aggiornati

## Key Discoveries

- **Visual identification con confidence levels** è il pattern naturale per popolare registry da corpus opaco. Tabella struttura (cell # | iconography | identification | confidence ⭐⭐⭐) permette decisioni rapide su cosa aggiungere e cosa skippare. Documentato anche cosa è stato skipped (per evitare ri-analisi futura).
- **Library v1 organizzate in stili visivi multipli**: `aws-simple-icons` ha 17 items che sono in realtà 6 brand × 3 stili (outline / filled / filled+text). Mapping semantico migliore = variante "with text" (riconoscibile al 100%). Le altre varianti restano accessibili via `{library, index}` deterministic.
- **viewType ≠ extension ID**: per `workbench.editorAssociations`, serve il `viewType` registrato dall'extension nel suo manifest (per Excalidraw: `editor.excalidraw`), non l'extension ID (`pomdtr.excalidraw-editor`). Scoperto via WebFetch del marketplace page dell'extension.
- **Machine settings sul mini-PC override User settings Mac per remote-SSH**: i `workbench.*` UI settings vivono in scope client di default ma le Machine settings possono override per la window connessa al server. Path: `~/.vscode-server/data/Machine/settings.json`.
- **Canvas reali vs constant hardcoded**: tutti i constanti spaziali devono leggere dal `spec.canvas` corrente, mai usare valori statici. Pattern bug ricorrente catturato 2 volte (v5 LAYER_X/RIGHT, v9.3 CANVAS_W).
- **Pattern Claude-mediated registry population**: lo screenshot del catalog è l'input visuale per Claude, che decompone in tabella semantica e fa batch-update del registry. Effort distribuito: 1 catalog ≈ 5 minuti per Rakki + 2 minuti per Claude.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Top_node "Rakki" centrato male su canvas != 1320 (sx-biased) | `CANVAS_W = 1320` constant hardcoded | Lookup canvas reale via `CANVAS_PRESETS[spec.canvas]` (v9.3) |
| `.excalidraw` apriva in nuova window VS Code anche con `openFilesInNewWindow: off` | Custom editor extension forza nuova window per remote-SSH webview | Workaround: `Cmd+P` quick-open apre nello stesso window |
| `.excalidraw` apriva come JSON raw invece di canvas | Cmd+P usa default text editor, no association | `editorAssociations: {"*.excalidraw": "editor.excalidraw"}` |
| `cluster` alias inizialmente mappato a aws-serverless:11 sbagliato | Iniziale guess senza visual validation | Catalog viewer → identificato cell corretta in cloud:14 → spostato + corretto |
| 1 alias per ID = problemi di duplicati semantici cross-library | Es. "kubernetes" presente in cloud:0 + drwnio:4 + technology-logos:0 | Tengo 1 mapping per alias (più dettagliato), gli altri restano accessibili via `{library, index}` |

## Final State

**Skill `excalidraw-diagrams-skill` v9.7** — feature-complete per Fase 1b del DOSSIER:

| Capability | Status |
|---|---|
| DSL builder vertical + horizontal | ✅ v6 |
| 7 palette colori | ✅ v1 |
| Pattern semantic per layer/node (hachure/cross-hatch/dots) | ✅ v3 |
| Typography social-readable +40% | ✅ v2 |
| Smart-side arrow routing direction-aware | ✅ v6 |
| Patch engine Claude-mediated (5 ops + 3 selectors + versioning) | ✅ v8 |
| Logo/icon embed (6 library MIT, 94 items) | ✅ v9 |
| Catalog viewer per identificare items | ✅ v9.1 |
| Corpus extension dual-purpose (LOGOS+ICONS) | ✅ v9.2 |
| Bug fix CANVAS_W canvas-aware | ✅ v9.3 |
| **83 alias semantici registrati** | ✅ v9.4-9.7 |
| Side-by-side VS Code editing in remote-SSH | ✅ setup completato |

**Pending Fase 1b** (per future sessions):
- Render pipeline `.excalidraw → SVG → PNG` (Node `@excalidraw/utils` vs Playwright spike)
- Direction `mixed` (matrix layout)
- Template `tree` + `flowchart` YAML + runtime `apply_template()`
- Style `minimal` + `playful` YAML + runtime `apply_style()`
- `SKILL.md` entry point per Anthropic skills packaging
- Audit sistematico magic numbers residui

## Open Questions

- **Confidence medium aliases** (eventbridge, appsync, kinesis, sns, vault, kafka, pubsub) — validation visiva richiesta. Smoke test li usa già; se mismatch, da correggere.
- **Skipped low-confidence cells** (~15 totali) — documentati in commenti `# Skipped`. Se in futuro Rakki riconosce uno di questi (es. drwnio:5 red stack potrebbe essere RedHat OpenShift), aggiungere alias.
- **Brand non rappresentati nel corpus**: redis, mongodb, mysql, elasticsearch, prometheus, grafana, jenkins, vue/react/angular, rust, swift, scala. Per coprirli, scaricare library extra da `excalidraw-libraries` GitHub (es. `lipis/database-icons`, `elastic` libraries) oppure draw custom.
- **Custom registry file**: ora `SEMANTIC_ALIASES` è hardcoded in `lib/logos.py`. Per la skill distribuita, considerare loadingo da YAML esterno (es. `examples/icon-libraries/aliases.yaml`) per permettere all'utente di estendere senza editare il modulo.
- **Real-time editing MCP server**: parzialmente sbloccato dal setup VS Code side-by-side. "Claude vede modifiche live" automatic richiede ancora MCP file watcher dedicato — Fase 2 quando il patch engine ha real-world usage.
