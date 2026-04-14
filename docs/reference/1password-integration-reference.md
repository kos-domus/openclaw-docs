---
title: "1Password Integration Reference"
slug: "1password-integration-reference"
category: "reference"
tags: ["1password", "secrets", "security", "reference"]
sources:
  - "sessions/2026-04-07-model-config-secret-caching-openclaw-update.md"
  - "sessions/2026-04-07-claude-code-agent-framework-mcp-fleet-fixes-2.md"
last_updated: "2026-04-14"
version: 1
---

# 1Password Integration Reference

## Problem pattern

Live `op read` calls on every startup are fragile in service contexts.

The processed sessions documented a failure loop:

1. service starts
2. startup script performs many sequential `op read` calls
3. 1Password rate limits the service account
4. startup fails
5. systemd restarts the service and repeats the loop

## Safer pattern

- keep the canonical live loader in `op-env.sh`
- generate `op-env-cached.sh` with resolved values
- source the cached file first during startup
- regenerate the cache when secrets rotate

## Operational requirements

- keep the cached file permissions locked down, for example `600`
- avoid committing rendered secret files
- document which services depend on the cached loader
- if a process spawns children, ensure the exported variables are inherited by the parent runtime

## When this matters most

- OpenClaw gateway restarts
- MCP server launches
- cron jobs or local services that must come up without interactive auth
