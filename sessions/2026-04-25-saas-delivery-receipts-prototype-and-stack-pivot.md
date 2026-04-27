---
title: "Saas Delivery Receipts: extraction prototype scaffolded + stack pivot from client doc"
date: "2026-04-25"
author: "kos-domus"
status: "processed"
tags: ["multi-agent", "automation", "api", "configuration", "troubleshooting"]
openclaw_version: ""
environment:
  os: "Linux 6.17.0-20-generic"
  ide: "VSCode (Claude Code extension)"
  model: "claude-opus-4-7[1m]"
---

## Objective

Two distinct things in one session:

1. Move the Saas Delivery Receipts (Sacchitalia DDT normalizer) project from "planning + architecture docs" to "running extraction harness scaffold, committed to git".
2. Read a planning document the client just shared (`PIANIFICAZIONE_CARICO_CARTA.md`) and produce a stack comparison between what the client proposes and what we had planned.

## Context

- Project lives at `~/job-desk/Saas Delivery Receipts/` (note: spaces in folder name — must be quoted in shell).
- Previous session (`2026-04-24-project-kickoff`) had produced: `SOUL.md`, 11-file co-CEO architecture in `docs/architecture/`, supplier-samples-analysis covering 7 real DDTs, and an empty src/ directory.
- Three gold files already hand-labeled by previous Kos work: BILLERUD (36 rolls), SMURFIT (27, with European thousand-separator + tonne weights), NATRON (27 across 3 product variants). The 7 supplier PDFs themselves are NOT yet in `samples/pdfs/` — they were shared in chat but not archived.
- Project carries a strict invariant: **product-agnostic architecture** — Sacchitalia is the first tenant, not the model.

## Steps Taken

### 1. Verified existing scaffold

`ls -la` on the project showed the previous session had left in place: `package.json`, `tsconfig.json`, `.gitignore`, `schema/`, `src/`, `docs/`, `sessions/`, `samples/`, plus `SOUL.md` and `DOSSIER.md`.

**Result**: Scaffold tree intact; only missing piece was `node_modules/` and `dist/`.

### 2. Installed dependencies

```bash
cd "/home/kos/job-desk/Saas Delivery Receipts" && npm install
```

**Result**: Added 8 packages (ajv, typescript, @types/node + transitive). 0 vulnerabilities.

### 3. First TypeScript build — failed on AJV interop

```bash
npm run build
```

**Result**: Compile error in `src/evaluate.ts`:

```
src/evaluate.ts(85,19): error TS2351: This expression is not constructable.
  Type 'typeof import("...ajv/dist/ajv")' has no construct signatures.
```

Cause: AJV v8 is shipped as CommonJS with `module.exports = Ajv; module.exports.default = Ajv; module.exports.Ajv = Ajv;`. Under TypeScript with `"module": "NodeNext"` + `"type": "module"` in `package.json`, the default import resolves to the namespace object, not the class.

### 4. Fixed AJV import with the standard ESM/NodeNext workaround

`src/evaluate.ts:13-14`:

```ts
import _Ajv, { type ErrorObject } from 'ajv';
const Ajv = _Ajv as unknown as typeof _Ajv.default;
```

**Result**: `npm run build` clean. `dist/` populated with `evaluate.js`, `extract.js`, `prompt.js`, `types.js` and source maps.

### 5. Updated DOSSIER.md milestone log

Added a 2026-04-24 PM entry to "Cosa è stato costruito" describing the prototype harness (schema/ + src/prompt.ts with 10 normalization rules + src/extract.ts using pdftoppm@150dpi + Gemini 2.5 Flash + src/evaluate.ts with AJV + per-field tolerances ±2mm/±1kg/±5mm). Also expanded "File chiave" section.

**Result**: DOSSIER reflects the new milestone; future sessions reading the dossier first will see scaffolded state, not "zero codice ancora".

### 6. Initialized git and committed scaffold

```bash
git init -q
git add .gitignore SOUL.md DOSSIER.md package.json package-lock.json \
        tsconfig.json schema src docs sessions samples
git commit -m "Initial commit: Sacchitalia DDT extraction harness scaffold"
```

**Result**: Commit `be96e7c`. `.gitignore` correctly excludes `node_modules/`, `dist/`, `samples/pdfs/*.pdf` (client data), `samples/outputs/*.json` (regenerated each run). Gold JSONs ARE committed because they are the public expected-output labels.

### 7. Client shared planning document — produced stack comparison

User pasted `PIANIFICAZIONE_CARICO_CARTA.md` from the client. Key revelations the doc made explicit:

- The product is NOT a multi-tenant SaaS — it is a bespoke integration with the client's existing ERP **ESOLVER**, deployed on their existing Node/Express server **JSAPP**.
- Final sink of the pipeline is **REST API ESOLVER** (currently "in attesa" — main blocker).
- Staging table `SI_CARICHI_BOZZA` on SQL Server with state machine BOZZA → CONFERMATO → IMPORTATO.
- Normalization driven by an `FORNITORI.XLSX` Excel file editable by employees (zero deploys to add new suppliers).
- Client proposes Tesseract.js OCR + pdf-parse + Claude Sonnet 4.6 for extraction.

I produced a comparison and pushed back on one specific decision: **Tesseract OCR vs vision model**. We learned painfully on Spesabot that Tesseract on real-world scans (rotated, photocopied, table-heavy) produces garbage that no LLM can rescue. Vision-first extraction (image → vision LLM → structured JSON) keeps far more signal.

**Result**: User parked the decision — will speak with the client in the morning and bring back answers. No code changes from this comparison yet.

## Configuration Changes

- New git repo at `~/job-desk/Saas Delivery Receipts/.git`
- `node_modules/` populated (gitignored)
- `dist/` build artifacts present (gitignored)
- DOSSIER.md updated with PM milestone

## Key Discoveries

- **AJV v8 + `"module": "NodeNext"` + `"type": "module"` requires the `_Ajv` rename trick.** The `import Ajv from 'ajv'` pattern that works under CommonJS/`esModuleInterop` fails under NodeNext because Node treats AJV's CJS `module.exports` object as the default export. Fix: `import _Ajv from 'ajv'; const Ajv = _Ajv as unknown as typeof _Ajv.default;`. Worth filing under "ESM gotchas" — the same pattern will hit any CJS lib that doesn't ship a proper `exports` field.
- **The client's planning document is the most important artifact yet produced for the project.** It reframes the project shape from "multi-tenant SaaS" to "single-tenant ESOLVER integration on JSAPP server". Several pieces of our co-CEO architecture (Postgres + RLS + multi-tenant JSON schemas) become not-needed; others (vision-based extraction, Italian-language prompt rules) become more important. The DOSSIER explicitly invariant "tutto il prodotto resta agnostico rispetto al tipo di merce" survives — but only as an internal pattern, not as deployed multi-tenancy.
- **Excel-driven supplier mapping (`FORNITORI.XLSX`) is genuinely the right product choice.** Warehouse staff add new suppliers without devs being involved. We had not specified this in our architecture; we should adopt it.
- **The real blocker for v0 is REST API ESOLVER access, not extraction accuracy.** If that slips, we should still build Modules 1–3 (read + extract + staging review) as a standalone CSV/Excel-export tool so we don't sit idle.
- **Tesseract OCR is the single technical mistake in the client's plan.** Worth fighting for — everything else is fine or negotiable.
- **IDE diagnostics can be stale relative to actual build state.** After running `npm install`, the IDE still reported "Cannot find name 'process'" — but `tsc` compiled clean. Trust the compiler over the IDE squiggles when there's a conflict immediately after dependency install.

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `TS2351: This expression is not constructable` on `new Ajv(...)` | AJV v8 CJS default-export interop with NodeNext+ESM resolves to namespace object, not class | `import _Ajv from 'ajv'; const Ajv = _Ajv as unknown as typeof _Ajv.default;` |
| Stale IDE diagnostics ("Cannot find name 'process'/'node:fs'") after the edit | IDE TS server cached pre-`npm install` state | Ignored — verified via actual `npm run build` which was clean |

## Final State

- `~/job-desk/Saas Delivery Receipts/` is now a real git repo at commit `be96e7c`.
- Build is green. Extraction harness scaffolded end-to-end: `pdftoppm` rasterize → Gemini 2.5 Flash vision → JSON output → AJV+comparator evaluation against gold files.
- DOSSIER.md current as of session end. Future sessions reading the dossier first will see "scaffolded prototype awaiting PDFs", not "zero codice".
- Architectural pivot is on the table but unresolved — waiting on client conversation tomorrow morning.
- Conversation switched at end: user asked to save this session and open a new one to check Spesabot status.

## Open Questions

- **For the client (Sacchitalia)**:
  1. Is REST API ESOLVER access scheduled, or open-ended "in attesa"?
  2. Hard requirement for Claude Sonnet 4.6 specifically, or is Gemini 2.5 Flash acceptable? (~10× cost difference at volume.)
  3. Does JSAPP have egress for external LLM APIs, or do we need a proxy?
  4. Realistic daily DDT volume?
  5. `SI_CARICHI_BOZZA` schema target + ESOLVER REST API spec/sandbox — when can we see them?
- **For Alessandro**:
  - Reply to client in Italian with the 5 questions above, or ask the co-CEO sub-agent to produce a formal architecture-pivot document reconciling our plan with the client's? (Recommended order: questions first, pivot doc only after answers.)
- **Project-level**:
  - 4 gold files still to label: CANFOR, GASCOGNE, HORIZON, IGUACU.
  - 7 supplier PDFs need to land in `samples/pdfs/` before the harness can produce its first real accuracy number against the v0 ≥80% gate.
