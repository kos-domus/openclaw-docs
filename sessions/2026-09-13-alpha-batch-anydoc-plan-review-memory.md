---
title: "Alpha batch 8 tool: adozione Anydoc (CSO-gated), cross-provider plan review in /plan (pattern claudex-loop), regole memory-system"
date: "2026-09-13"
author: "kos-domus"
status: "new"
tags: ["security", "cli", "skills", "agent-sdk", "workflow"]
session_type: "claude-code"
openclaw_version: ""
environment:
  os: "Linux (mini-PC)"
  ide: "Claude Code (VSCode extension)"
  model: "claude-fable-5-1"
---

## Objective
Valutare 8 tool/risorse (Claudex Loop, Anydoc, DeepSeek Harness, Orca, FunASR, prompt "Project Memory & Decision System", Audiobookshelf, fx) con lente `/alpha`; poi, su decisione utente, adottare Anydoc, rubare il pattern claudex-loop dentro `/plan`, e prendere dal memory system ciò che serve.

## Steps Taken
### 1. Valutazione (clone inerti in `_eval-quarantine/alpha-2026-09-13/`)
Verdetti: Anydoc ADOTTA-gated · Claudex STEAL-pattern · aiedge #08 2 micro-steal (NON incollare il prompt in Hermes) · DeepSeek Harness / Orca / FunASR / Audiobookshelf / fx SKIP motivati (dettaglio MOC Agentic-Engineering §Batch 2026-09-13).
**Result**: nessun install in fase di valutazione; CSO precheck su Anydoc → GO-with-conditions C1–C6.

### 2. Adozione Anydoc
- Cargo assente sul mini-PC → C1 fallback: venv `~/.local/share/anydoc/venv`, `uv pip install --require-hashes` (8 hash PyPI, 0.2.4). trivy rootfs: 0 vuln.
- CLI nostro `anydoc_cli.py` (nessuna opzione hosted) + wrapper `~/.local/bin/anydoc`: rifiuta `--ocr/--api-key/--api-url` (exit 2), `unset FIRECRAWL_*`, `unshare -Un` + `systemd-run --user --scope MemoryMax=2G` + `timeout 120s`.
- Skill `~/.claude/skills/anydoc/SKILL.md`; register `sacchitalia/.security/vendor-tools-register.md`.
**Result**: docx/pdf/csv OK; PDF scansionato → exit 3. **Scoperta**: anydoc (pdf-inspector) non legge il text layer invisibile di ocrmypdf → il percorso locale per gli scansionati è `ocrmypdf --skip-text --sidecar OUT.txt` (o `pdftotext -layout`), non ri-lanciare anydoc.

### 3. Cross-provider plan review in /plan
Runner `~/.claude/skills/plan/scripts/plan-review.py` (stdlib): `codex exec -s read-only --output-schema`, verdetto APPROVED/REVISE/BLOCKED, approvazione legata a SHA256, log append-only, round cap 3, output vuoto ≠ approvazione. Fase 3b in SKILL.md.
Ostacoli risolti: (a) snap Codex 0.114 non legge dot-dir né /tmp e non supporta gpt-5.5 → installato `@openai/codex@0.154.0` npm user-scope (precede lo snap); auth resta `apikey` (sk-proj a pagamento), **mai `codex login`** (contenderebbe l'OAuth di Hermes mc). (b) `kernel.apparmor_restrict_unprivileged_userns=1` rompe bubblewrap → `--enable use_legacy_landlock` (deprecato upstream; override `PLAN_REVIEW_SANDBOX_FLAGS`), verificato che blocca `touch`.
**Result**: smoke test su piano giocattolo → REVISE con finding reale (grep -qx non verifica l'exit code dello script); `check` SHA-binding OK.

### 4. Memory system (aiedge #08)
- `reference_memory_edit_strategy_by_layer`: nuovo layer *decisions → supersede-preserved*.
- Nuova memoria `feedback_recall_past_decisions` ("l'ultima volta" → grep decision memo/plan/session log, cita, esponi Superseded).
- `/plan` Fase 4: memory audit a milestone (5 check, incluso fresh-session resume test).

## Lessons Learned
- Il Codex CLI snap è confinato: path visibili sotto $HOME o fallisce silenziosamente con "Permission denied" sullo schema.
- Su Ubuntu recente il sandbox bwrap di Codex non parte senza allentare AppArmor; Landlock è l'alternativa senza toccare la posture host.
- Le skill upstream dei vendor possono istruire l'agent a fare egress (`--ocr hosted`): riscrivere sempre la skill in forma nostra dopo il CSO gate.
- Un errore Codex "not supported with a ChatGPT account" può apparire anche in `auth_mode: apikey`: non dedurre la modalità auth dal messaggio, leggere `auth.json` (solo chiavi).

## Files Touched
`~/.local/share/anydoc/{requirements.txt,anydoc_cli.py,venv}` · `~/.local/bin/anydoc` · `~/.claude/skills/anydoc/SKILL.md` · `~/.claude/skills/plan/SKILL.md` · `~/.claude/skills/plan/scripts/plan-review.py` · `~/Obsidian-Personal/MOC/Agentic-Engineering.md` · memorie job-desk: `reference_community_claude_resources`, `reference_memory_edit_strategy_by_layer`, `feedback_recall_past_decisions`, `MEMORY.md` · `sacchitalia/.security/vendor-tools-register.md` · `_eval-quarantine/alpha-2026-09-13/cso-precheck-anydoc.md`.
