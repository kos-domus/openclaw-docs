---
title: "Cron and Scheduling Reference"
slug: "cron-scheduling-reference"
category: "reference"
tags: ["cron", "scheduling", "automation", "reference"]
sources:
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
last_updated: "2026-04-14"
version: 1
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

## When to choose systemd instead of OpenClaw cron

Use systemd timers when you need:

- strict resource caps
- one-job-per-source isolation
- cooldowns between tasks
- local daemon-style orchestration

That was the recommendation for the SpesaBot ingest runner.
