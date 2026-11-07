---
title: "Fase V: Reference-plus-Mutants Verification with Cross-Provider Review"
slug: mutant-testing-cross-provider-review
category: guides
tags: ["testing", "mutant-testing", "verification", "cross-provider", "side-effects"]
sources:
- sessions/2026-11-06-decisore-locale-spesabot-pipeline-fase-v.md
last_updated: "2026-11-07"
version: 1
---

# Fase V: Reference-plus-Mutants Verification with Cross-Provider Review

A repeatable method for verifying a small automation script — here, a downloader against an unofficial REST API (login, listing, "open" an item, fetch attachments via CDN) — without trusting the script's author, the implementing model, or any single reviewer.

## The method

1. **Fake infrastructure, not mocks-in-code.** Stand up three local servers: a fake API, a fake CDN, and a *foreign origin* plus a proxy pointed at it. The script under test cannot tell it is not talking to the real service.
2. **A reference implementation written by the planner** (not the implementer) defines expected behavior. The reference must pass everything: `TUTTO OK`.
3. **N deliberately wrong variants ("mutants")** — 44 in the observed run — each break one behavior: wrong headers, wrong destination, missing check, touching the foreign origin, etc.
4. **The verifier must catch 44/44 mutants.** A mutant that escapes means the test bench is blind somewhere, not that the mutant was subtle.

> One variant escaped because HTTP headers were asserted in only one scenario. Assert headers **globally across every request the script makes**, not per-scenario.

## Cross-provider review with an explicit human gate

The verification script itself was then reviewed by a *different* provider than the one that wrote it. The review rejected the script **twice**. The working rule:

- After the second rejection, the **human decides** — an unbounded review loop is its own failure mode.
- Fix the real defects found; explicitly declare the remaining objections out of scope; proceed with a **registered, explicit override**.
- The override record is part of the artifact: who decided, what was waived, when.

## Confining the "read" side effect

An unofficial API that returns 404 for attachments until the item has been "opened" forces a design decision: fetching an attachment *marks it read* on the remote service. Rules that kept this safe:

- the side effect is surfaced to the user and **decided by the user**, once, up front
- execution is confined: a single fixed path, a per-run cap, never repeated in a loop
- the executor runs without access to the test bench (graph separation between "can execute" and "can verify")

## Why the separation matters

Planner writes the reference, implementer writes the script, a different provider reviews, the verifier only verifies, the executor only executes. Every artifact (mutants, review rounds, override) is written down. The method caught every seeded defect and survived two hostile reviews — the failures it produced were about *scope*, which is exactly the argument a record should settle.
