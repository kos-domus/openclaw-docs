---
title: "Unified Agent Memory: Federate First, Consolidate Later"
slug: unified-agent-memory
category: concepts
tags: ["memory", "multi-agent", "mcp", "sqlite", "fts5", "security", "design"]
sources:
- sessions/2026-10-27-deco-backhaul-printer-audio-memory-plan.md
last_updated: "2026-10-27"
version: 1
---

# Unified Agent Memory: Federate First, Consolidate Later

Design for a shared memory layer across all agents (Claude Code instances and Hermes profiles, later a read-only desktop client). The plan is in **DRAFT** — nothing executed yet — but the design decisions are the durable knowledge.

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
