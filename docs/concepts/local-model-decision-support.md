---
title: "Local Decision Models: What Small LLMs and Embeddings Can and Cannot Do"
slug: local-model-decision-support
category: concepts
tags: ["llm", "embeddings", "llama-cpp", "benchmarks", "evaluation", "security"]
sources:
- sessions/2026-11-06-decisore-locale-spesabot-pipeline-fase-v.md
last_updated: "2026-11-07"
version: 1
---

# Local Decision Models: What Small LLMs and Embeddings Can and Cannot Do

Can a fully local stack — a small LLM on CPU via llama.cpp plus a local embedding model — autonomously filter review points in a cross-provider plan revision? A labeled evaluation set of 331 review points (criteria G1–G6) says: **no for autonomous filtering, yes as a candidate ranker.** The division of labor is the durable finding.

## The core insight: contradiction is not similarity

"Reopens a decision that was already made" (G1) is a *semantic contradiction*, not a textual similarity. This has a direct consequence for tool choice:

- **An embedding model finds things that look alike.** Contradiction does not look like anything — it is a relation between meanings, not a surface form.
- **A small CPU LLM can classify by area** (e.g. "is this a security concern?") but cannot reliably detect that two statements conflict.

G1 therefore stays with the human reviewer or a large model. Local components were measured, not assumed:

## Measured results (331 labeled points)

| Approach | Result |
|----------|--------|
| Small LLM, multiple-choice over a 58-entry decision log (G1) | correct entry 0/9, AUC 0.42 — probability smeared across options; ~263 s per point on CPU |
| Local embedding (G1, with safety gate) | best AUC 0.65, correct entry top-1 3/9 |
| Embedding as duplicate *search* (G5, 101 known duplicates) | threshold yes/no AUC 0.70 — but as ranked retrieval: original in top-5 for 91% of points, top-1 for 61% |
| Two-stage: embedding proposes top-3 pairs, LLM confirms (969 pairs, ~28 s/pair) | AUC 0.78; recall doubles vs embedding alone at precision ~0.7; correct top-1 57/74 vs 54/74 |

The two-stage combination is real progress, but still below a 0.90 precision bar for unattended filtering. Conclusion: use it to **order candidates for a human**, never to auto-dismiss.

## Operating model for local model serving

Small models on a shared home server need explicit containment. The review-approved setup:

- llama.cpp built from source, `llama-server` bound to **127.0.0.1 only**
- typed API surface — question kinds (`noul`/`choice`) rather than free-form generation
- official GGUF releases only, sha256 pinned in a `SHA256SUMS` file
- `--no-slots`, service runs as a dedicated sandboxed user unit
- models requiring a local torch-based conversion were **deferred** — conversion pipelines expand the trusted surface for marginal gain

## Practical sizing notes

- HTTP 500 on every benchmark point was caused by input exceeding the 4,096-token batch default — launch with `-c 16384 -b 8192 -ub 8192` and stop after 3 consecutive errors instead of burning a run on a broken server.
- A review round counter that counts rounds already in the log will silently skip round 2 — pass the intended count explicitly (`--rounds 2`).

## Where this goes next

Same local stack, different questions where similarity *is* the signal: mail triage by area, and product matching across supermarket chains — where the open question is whether the pair can cross a 98% precision bar.
