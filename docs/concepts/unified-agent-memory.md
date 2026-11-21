---
title: "Unified Agent Memory: Federate First, Consolidate Later"
slug: unified-agent-memory
category: concepts
tags: ["memory", "multi-agent", "mcp", "sqlite", "fts5", "security", "design"]
sources:
- sessions/2026-10-27-deco-backhaul-printer-audio-memory-plan.md
- sessions/2026-11-21-memoria-release-a-k2-firecrawl-mcp.md
last_updated: "2026-11-21"
version: 2
---

# Unified Agent Memory: Federate First, Consolidate Later

Design for a shared memory layer across all agents (Claude Code instances and Hermes profiles, later a read-only desktop client). **Release A is installed and verified (2026-11-21)**; the sections below record the design decisions and the installation protocol that got it there.

## The Problem

Memories already exist in five places: Claude's own memory files, each Hermes profile's `MEMORY.md`, project dossiers, and an Obsidian vault. Two consequences:

1. No agent can answer "what do we know about X" across the whole fleet.
2. **A containment gap:** the pre-plan inventory found *work* agents that could read *family* data through their MCP filesystem roots. The first step of a unified memory is not federation — it is **containment**.

## Core Architecture

**Federate first, consolidate later.** No migration. A SQLite FTS5 indexer reads memories *where they already live* and builds one searchable index over them. Sources of truth stay untouched; the index is disposable and rebuildable.

On top of the index, an **MCP stdio "memory" server** with **compartments fixed at startup** — home, work, personal, mail, shared. Each agent is wired to see only its own compartments. Tools:

`search` · `get` · `recent` · `propose` · `supersede`

## Mutation Strategy by Record Level

Not everything should be editable the same way:

| Level | Strategy |
|---|---|
| Facts | **ADD-only**, corrections via `superseded_by` pointer — history preserved |
| Rules | edited in place |
| Decisions | replaced but never deleted — the old text stays linked |

This makes memory append-mostly, which is both auditable and safe for concurrent agents.

## Why FTS5 Before Embeddings

On a small memory corpus in Italian, BM25 search (SQLite FTS5) beat small local embeddings in an earlier test: **73% vs 46%**. The plan starts with FTS5 and requires measurement *before* adding an embedding layer. Small corpora punish semantic drift more than they reward fuzzy matching.

## The Eight Units

Sequenced smallest-safe-step-first, with **mandatory security review before U3, U6, and U7**:

- **U0 — Containment:** family directories removed from work agents' MCP filesystem roots. (Immediate, no new infrastructure.)
- **U1 — Schema and compartment map.**
- **U2 — Indexer**, validated with 20 golden questions, target **recall@5 ≥ 85%**.
- **U3 — MCP server.**
- **U4 — Per-role wiring** (which agent sees which compartments).
- **U5 — Deduplication.**
- **U6 — Hardening:** one Unix user per agent.
- **U7 — Remote read-only access** — last, only after everything else holds.

## Release A — Installation Record (2026-11-21)

Release A shipped through a guided, evidence-first protocol. The durable part is the protocol itself — it generalizes to any "install generated code into production" event:

1. **Go/no-go with re-run verifications** — all 8 checks re-executed against the code before install (8/8 green), with the **input fingerprints of the scripts unchanged** since review. Fingerprinting the inputs is what makes a re-run a real re-verification instead of a re-roll.
2. **Install from a frozen copy owned by root**, checksummed (`sha256` of the manifest) against a value **communicated out-of-band in chat** — never read from the writable file itself. A checksum stored next to the artifact it validates protects nothing.
3. **Rollback ready**: a copy of the previous release kept before the swap, `cmp` after install to prove the deployed tree matches the reviewed one, rebuild, verifications green again.
4. **Before/after measurement**: 10 questions answered by both versions, judged blind — **no regression**. This step is what caught a real problem: the "after" fingerprints matched a measurement taken two days earlier → production files had been written by *not-yet-installed* code (stale automatic executors that could still launch Bash). Measure with input fingerprints; a naive before/after would have missed it.
5. **Human gate before install**: the rule titles agents would see as "approved" were confirmed by the operator *before* the release went in, not after.

Related hardening from the same cycle: [DB Credential Hardening](../guides/db-credential-hardening.md). Still open from the session: the umask of agent temp files (world-readable on every new report) and locking down the plan executor so it cannot launch Bash or write production files.

## Open Decisions (blocking execution)

- Does the consolidated store eventually live in the Obsidian vault?
- Does the home agent see part of *personal*?
- Scope of remote access (U7)?
- One Unix user per agent — confirmed as design, details pending?
- Do Claude's own memories remain a personal index rather than merging in?

## Related

- [Workspace Isolation](workspace-isolation.md) — the filesystem-level separation this plan builds on.
- [Semantic Data Layer](semantic-data-layer.md) — the earlier entity-graph approach to shared data.
- [MCP Servers](../reference/mcp-servers.md) — transport and configuration for the memory server.
- [Multi-Agent Architecture in OpenClaw](multi-agent-architecture.md) — fleet topology the compartments map onto.
