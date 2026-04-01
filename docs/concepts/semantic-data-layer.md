---
title: Building a Semantic Data Layer on Google Drive
slug: semantic-data-layer
category: concepts
tags:
- database
- google-drive
- json
- entity-graph
- architecture
sources:
- sessions/2026-03-28-familyos-database-kai-1password.md
last_updated: '2026-03-30'
version: 2
---

# Building a Semantic Data Layer on Google Drive

This document describes the architecture for building an AI-queryable semantic database on Google Drive — using JSON files, entity linking, and an entity graph to give agents structured access to data without a traditional database.

## Why Google Drive Instead of a Database?

For personal and family-scale use cases:

| Factor | Google Drive | Traditional DB |
|--------|-------------|---------------|
| Accessibility | Browse from any device (phone, browser) | Requires client or API |
| Versioning | Built-in revision history | Must configure separately |
| Cost | Free (15 GB) | Hosting costs |
| Backup | Google handles it | Your responsibility |
| Agent access | Via GWS CLI (already configured) | Additional integration |
| Human browsability | Native (folders, files) | Requires UI |

**Trade-off**: Drive is not suitable for high-volume transactional workloads. For personal data management (thousands of records, not millions), it works well.

## Design Principles

### 1. JSON-First

- **Structured data = JSON** (queryable by agents)
- **PDFs, images, scans = archival** (stored alongside, referenced in JSON)
- When processing a receipt or document, extract data into JSON and store the original as archival

### 2. Entity Linking

Every piece of data **must** reference at least one of:
- `person_id` — who it's about
- `category` — what type it is
- `date` — when it happened

This enables cross-entity queries: person → expenses → activities → contacts.

### 3. Single Source of Truth

- One authoritative file per entity type (e.g., `master_contacts.json`)
- Category subfolders are **logical views**, not data stores
- Never duplicate data across files

### 4. Consistent ID Patterns

| Entity | Format | Example |
|--------|--------|---------|
| Person | `{name}` | `kos`, `katia` |
| Expense | `exp_YYYYMMDD_NNN` | `exp_20260315_001` |
| Contact | `ct_lastname_firstname` | `ct_smith_john` |
| Event | `evt_YYYYMMDD_shortname` | `evt_20260401_dentist` |
| Activity | `act_category_shortname` | `act_sports_swimming` |

## Example Folder Structure

```
My Drive/Data/
├── 01_Members/           → profile.json per person (identity nodes)
├── 02_Finance/           → expenses (raw + processed), reports
├── 03_Contacts/          → master_contacts.json (single source of truth)
├── 04_Schedule/          → events, recurring rules
├── 05_Activities/        → linking hub (costs, contacts, events)
├── 06_Documents/         → legal, insurance, identity
├── 07_Health/            → per-person visits, vaccinations
├── 08_Knowledge/         → preferences, routines, decisions
├── 09_Assets/            → subscriptions, devices
├── 10_Travel/            → trips, documents, expenses
└── 99_System/            → schemas, entity graph, agent config
```

## The Entity Graph

The entity graph (`entity_graph.json`) is the key to making this queryable. It defines:

1. **Relationships** between entity types with cardinality
2. **Source file patterns** for locating data
3. **Pre-built query patterns** for common operations

```json
{
  "entities": {
    "person": {
      "source_pattern": "01_Members/{person_id}/profile.json",
      "relationships": {
        "expenses": { "type": "one_to_many", "target": "expense", "via": "person_ids" },
        "activities": { "type": "many_to_many", "target": "activity", "via": "person_ids" },
        "events": { "type": "one_to_many", "target": "event", "via": "person_ids" }
      }
    }
  },
  "query_patterns": {
    "expenses_by_person_and_month": "SELECT * FROM expenses WHERE person_id = ? AND month = ?",
    "activity_total_cost": "SELECT SUM(cost) FROM expenses WHERE activity_id = ?"
  }
}
```

**Key insight**: For AI agents, the relational graph matters more than the directory structure. The same data can be queried by person, by category, by date, or by activity — folders are a human convenience layer.

## Memory and Learning Layer

To enable agents that learn across conversations, add a memory system:

### Memory Types

| Type | When to Save |
|------|-------------|
| `correction` | User corrects the agent |
| `preference` | Agent learns how someone prefers things |
| `pattern` | Agent detects recurring behavior in data |
| `decision` | A decision worth remembering |
| `observation` | Agent notices something but doesn't act yet |
| `feedback` | User tells the agent to do something differently |

### Confidence Scoring

- Explicitly stated facts: confidence `1.0`
- Inferred patterns: confidence `0.5–0.8`
- Adjust over time as patterns are confirmed or contradicted

### Memory Rules

1. **Append-only**: Never delete memories — set `active: false` to invalidate
2. **Corrections supersede**: New memories can reference and invalidate old ones via `supersedes` field
3. **Read before acting**: Load memories at conversation start
4. **Save immediately**: Don't batch — append as you learn

## Adaptive Anomaly Detection

Instead of fixed thresholds ("alert if expense > $500"), use **rolling baselines**:

```json
{
  "spending_baselines": {
    "groceries": {
      "rolling_3month_avg": 450,
      "alert_threshold_pct": 30
    }
  }
}
```

Alert when a category exceeds its rolling 3-month average by 30%. This automatically adapts to seasonal changes (higher heating bills in winter, higher food costs during holidays).

## What's Next

- [Agent Fleet Design](agent-fleet-design.md) — Agents that operate on this data layer
- [Google Workspace CLI](../guides/google-workspace-cli.md) — How agents access Drive
