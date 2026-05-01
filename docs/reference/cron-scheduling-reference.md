---
title: "Cron and Scheduling Reference"
slug: "cron-scheduling-reference"
category: "reference"
tags: ["cron", "scheduling", "automation", "reference"]
sources:
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
  - "sessions/2026-04-30-fleet-fixes-spesabot-consolidation-esselunga-image-registry.md"
last_updated: "2026-05-01"
version: 2
---

# Cron and Scheduling Reference

## OpenClaw cron principles

- keep prompts explicit
- prefer agent defaults over hardcoded model overrides unless quality truly requires it
- design recurring jobs to SKIP cleanly when there is no new work
- remember that approval-dependent `exec` steps can dead-end in unattended runs

## Common knobs

| Setting | Meaning |
|---|---|
| cron expression | recurring schedule |
| timezone | schedule evaluation zone |
| session mode | isolated or shared behavior |
| announce | whether to deliver results back to a channel |
| channel | destination surface |
| model override | only when the default is not enough |

## Reliability lessons from the sessions

- Hardcoded invalid models in job payloads can break cron health across the system.
- Docs elaboration quality can drop if the job falls back to a cheaper model unexpectedly.
- Systemd timers are still the right choice for some heavyweight local pipelines, especially when they need strict RAM controls or custom retry logic.

## Staggering rules for dependent jobs

Treat cron design as a dependency graph, not a list of isolated reminders. If job B reads files, notes, or reports created by job A, do not schedule both on the same minute.

Recommended rule of thumb:

- **producer first**
- **consumer 10-20 minutes later**
- **heavy LLM jobs separated from each other** when they share the same provider budget

This prevents three common failure classes:

1. provider burst / rate-limit contention
2. race conditions on generated files
3. false-negative troubleshooting where the downstream job looks broken but simply ran too early

## Bootstrap state outside the main cron job

Some recurring jobs depend on files that are optional but noisy when missing, such as `memory/<today>.md`. If the runtime tolerates empty files, pre-create them with a small systemd timer or wrapper script before the main agent reset window instead of teaching every cron prompt to defend against `ENOENT`.

That pattern keeps the chat-facing jobs focused on real work while low-value housekeeping stays outside the agent loop.

## When to choose systemd instead of OpenClaw cron

Use systemd timers when you need:

- strict resource caps
- one-job-per-source isolation
- cooldowns between tasks
- local daemon-style orchestration

That was the recommendation for the SpesaBot ingest runner.
