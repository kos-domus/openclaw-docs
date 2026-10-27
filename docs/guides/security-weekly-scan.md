---
title: "Weekly Deterministic Security Scan for a Home Server"
slug: security-weekly-scan
category: guides
tags: ["security", "trivy", "osv", "exposure", "inventory", "cron"]
sources:
- sessions/2026-10-26-minipc-ops-printer-youtube-docker-security-scan.md
last_updated: "2026-10-27"
version: 1
---

# Weekly Deterministic Security Scan for a Home Server

A weekly `security-weekly` check built on one principle: **deterministic and boring on purpose** — no LLM in the loop, no auto-update, every step reproducible. It answers three questions: what is installed, what is vulnerable, what is exposed.

## 1. Inventory with Weekly Diff

Enumerate every software source on the host, then diff week-over-week so *changes* pop out instead of drowning in a full listing:

- `apt` packages, `snap` packages
- global npm (`npm -g`), `uv`/`pipx` tools
- **every agent's Python venv** — derived not only from workspaces but also from the `ExecStart=` lines of systemd units (services running code from venvs that no workspace listing reveals)
- Docker images
- Firefox extensions
- Home Assistant custom components

> **Gotcha:** scanning only workspace directories misses Python environments that exist purely as service targets. Parse `systemctl cat <unit>` for the interpreter path.

## 2. Vulnerability Scanning with Trivy

Keep Trivy itself current, but **verify the download before trusting it**: check the binary's SHA-256 against the release's `checksums.txt` before replacing the installed one.

Scan targets:

- a **minimal rootfs** containing only `os-release` and a copy of `dpkg/status` — this scans OS packages *without* walking the data disks;
- the package directories of the tools above;
- Docker images.

Filter to **HIGH and CRITICAL with an available fix** — severity without fixability is noise for a home server.

## 3. Malicious-Package Check via OSV

Use the OSV `querybatch` endpoint against all npm/PyPI packages from the inventory, looking specifically for **`MAL-*` advisory IDs** (OSV's malicious-package namespace) rather than plain CVEs.

Validate the detection path with a known-bad sample (e.g. `MAL-2022-1`) so a silent API change surfaces as a test failure, not as "zero findings".

## 4. Exposure Audit with `ss`

Group listening sockets by port and service, then exclude:

- loopback listeners,
- Docker internal networks,
- **ephemeral ports ≥ 32768**.

Without the ephemeral filter the report showed **61 "services"**; after filtering, **19 real ones**. Roughly two thirds of a naive exposure report can be client-side ephemeral sockets.

## 5. Reporting Cadence

- **Sunday:** full report to Telegram.
- **Daily:** alerts only for *new* findings — anything already reported stays in the weekly summary.

## First-Run Results (reference baseline)

- 0 malicious packages (npm and PyPI);
- 231 HIGH/CRITICAL vulnerabilities with fixes, immediately **grouped by who can fix them**: ours (uv/pipx tools, old kernels — fixed on the spot), upstream projects' (e.g. Fastify/@fastify/static in an exposed API, certifi/aiohttp in Python environments), vendor's.

That grouping is the operational point: it turns a scary total into three actionable buckets.

## Test Hygiene

- Never use `cat` on secret files inside test scripts — assert readability with `test -r` or `stat` instead.

## Related

- [Home Server Container Updates](home-server-container-updates.md) — the update procedure the scan feeds into.
- [Cron Jobs, Sub-Agents, and Automation](cron-jobs-and-automation.md) — timer wiring for the weekly/daily cadence.
