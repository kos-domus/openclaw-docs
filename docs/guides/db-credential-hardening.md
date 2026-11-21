---
title: "Hardening Database Credentials on a Multi-Agent Host"
slug: db-credential-hardening
category: guides
tags: ["security", "postgres", "credentials", "mcp", "hardening", "secrets"]
sources:
  - "sessions/2026-11-21-memoria-release-a-k2-firecrawl-mcp.md"
last_updated: "2026-11-21"
version: 1
---

# Hardening Database Credentials on a Multi-Agent Host

How to get database passwords out of places every local user can read: process argument lists, MCP config files, and world-readable temp files. Distilled from a security-closure session on a shared mini PC running an agent fleet.

## The Problem: Passwords on the Command Line

Passing the full connection URL (with password) to `psql` or `pg_dump` puts the secret in the process argument list. Any user on the system can read it:

```bash
ps aux | grep psql        # password sits right there
```

Verify the exposure live — the same fact that makes it readable makes it checkable. While a slow query runs, read the kernel's own record of the command line:

```bash
cat /proc/<pid>/cmdline    # exact args of the running psql
```

## The Fix: PGPASSWORD, Not the URL

A shared helper script derives a **password-less URL** and exports the password only into `PGPASSWORD`:

- `PGPASSWORD` lives in the process **environment**, which only the same UID (and root) can read via `/proc/<pid>/environ`.
- The argument list stays clean, so `ps` reveals nothing.

Concretely: four scripts that previously embedded the URL in their `psql`/`pg_dump` invocation were switched to the common helper (`pg-senza-password.sh` pattern): helper resolves host/db/user, exports `PGPASSWORD`, and `exec`s the client with the bare URL.

## Rotate, Then Prove the Door Is Closed

1. Rotate with `\password` inside `psql` — the new password travels over the already-encrypted channel, never appearing in shell history or args.
2. Prove another local user **cannot connect** and **cannot read the environment** of the DB-running processes.

> **Gotcha — a found secret may already be dead.** During the sweep, a plaintext password turned up in two agents' config files. Before estimating damage, test it: this one was the *old* password, already rejected by the server. Exposure was real but stale. Always check validity before scoring the blast radius.

## Remove Credential-Carrying MCP Servers You Don't Use

An unused postgres MCP server was discovered: launched with unpinned `npx` (whatever version lands) **and** the DB password in its config. Removed from two Hermes profiles and from the workspace `.mcp.json`. Rule: every MCP server that holds a credential is standing attack surface — unused ones get deleted, not disabled.

## Disk-Wide Secret Search Without Timing Out

A naive "search the whole disk for this password" in Python reads every file and blows past the timeout. The working alternative:

```bash
grep -rlF -f /root/secret-file-0600 / --exclude-dir=proc --exclude-dir=sys ...
```

- `-F` fixed-string (no regex engine cost), `-f file` takes the pattern from a `0600`-permission file, targeted exclusions skip virtual filesystems.
- Keep the pattern file locked down — it *is* the secret.

## Related Gotchas from the Same Sweep

- `pgrep -fc <pattern>` counts the invoking shell itself when the pattern appears in its own command line — subtract one or match more precisely.
- Verifying a short-lived service (a 3-second dump job) with "watch the process" fails; force an observable window instead (e.g. `psql` with `pg_sleep`, then inspect `/proc/<pid>/cmdline` while it runs).

## Related

- [MCP Servers and Tool Integration](../reference/mcp-servers.md) — scope, launchers, and the remote-server allowlist pattern.
- [1Password Secrets Management](1password-secrets-management.md) — where long-lived secrets should live instead of config files.
- [Weekly Deterministic Security Scan](security-weekly-scan.md) — the recurring check that catches regressions of everything above.
