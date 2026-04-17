---
title: "Environment Variables Reference"
slug: "environment-variables"
category: "reference"
tags: ["environment", "configuration", "secrets", "1password", "mcp"]
sources:
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
  - "sessions/2026-04-08-deep-research-pipeline-phase1-build.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
  - "sessions/2026-04-08-deep-research-glm51-zai-integration-4.md"
  - "sessions/2026-04-16-spesify-pipeline-images-matching-ui.md"
last_updated: "2026-04-17"
version: 2
---

# Environment Variables Reference

This reference collects the environment-variable patterns that repeatedly mattered in the processed sessions.

## Core rule

If OpenClaw, MCP servers, cron jobs, or child processes need a secret, make sure the variable exists in the **parent runtime environment**. Shell startup files alone are not enough if the process is spawned non-interactively.

## Frequently referenced variables

| Variable | Purpose | Where it mattered |
|---|---|---|
| `OPENAI_API_KEY` | Raw OpenAI API access for fallback extraction | Deep research LLM fallback |
| `ZAI_API_KEY` | Z.AI coding endpoint access | Deep research GLM planning and extraction |
| `FIRECRAWL_API_KEY` | Firecrawl fetch fallback | Deep research MCP server |
| `GITHUB_PAT` | GitHub MCP authentication | Claude Code MCP setup |
| `RESEARCH_JOB_TIMEOUT` | Per-job timeout override | Deep research pipeline |
| gateway/service secrets | Channel, provider, and runtime secrets | OpenClaw startup and cron health |

## Loading strategy that worked

### Preferred

- keep live 1Password reads in `op-env.sh`
- generate a cached `op-env-cached.sh` file with resolved values
- source the cached file on startup for services that must come up reliably

### Why

Repeated `op read` calls during every restart caused rate-limit loops when combined with `Restart=always`.

### Service pattern that held up

For user services or non-interactive runners, a stable pattern was:

```bash
set -a
source ~/.openclaw/op-env-cached.sh
set +a
```

This keeps startup deterministic and avoids depending on interactive shell initialization. If the service still fails, check that the cached file contains the same variable names the app expects, not just equivalent values under different names.

## Non-interactive shell gotcha

A recurring problem was the `.bashrc` non-interactive guard:

```bash
case $- in *i*) ;; *) return;; esac
```

This means MCP server spawns and other non-interactive processes may never see exports defined later in `.bashrc`.

## Practical guidance

- Put critical service env in the actual process environment, not only in interactive shell setup.
- If `.mcp.json` references `${VAR}`, that variable must already exist for the spawning process.
- Keep secrets out of tracked JSON when possible. Use env expansion instead.
- Document which flows use subscription auth versus pay-per-token API keys.
