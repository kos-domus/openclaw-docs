---
title: "Updating Home Server Containers Safely (Immich, n8n, Home Assistant)"
slug: home-server-container-updates
category: guides
tags: ["docker", "containers", "home-assistant", "n8n", "immich", "backup", "rollback"]
sources:
- sessions/2026-10-26-minipc-ops-printer-youtube-docker-security-scan.md
last_updated: "2026-10-27"
version: 1
---

# Updating Home Server Containers Safely (Immich, n8n, Home Assistant)

A repeatable update pattern for Docker-hosted services on a home server: dump first, pin deliberately, prepare the rollback path, and verify breaking changes against the *real* configuration — not the changelog abstract.

## The Pattern

1. **Dump the database before touching anything** (Immich: full DB dump; migration took ~1 s, but the dump is the insurance).
2. **Back up the configuration directory without logs** — logs are large and useless for rollback.
3. **Retag the currently-running image** (e.g. `homeassistant:2026.5.4-old`) so the previous version stays pullable locally as an instant rollback target.
4. **Read the breaking-changes list against your actual config.** In the Home Assistant 2026.5.4 → 2026.10.2 update, none of the breaking changes applied to the running config — but that was verified per-item, not assumed.
5. **Pull the new image, recreate, verify.**

## Worked Examples

### Immich 3.3.0 → 3.3.5 (security release)

A security release (CA eMudhra distrust in the `certifi` chain). Database dumped first; migration completed in ~1 s. Security-driven patch releases on Immich are low-drama *because* the dump exists.

### n8n 2.23.1 → 2.44.6, pinned by digest

**Why pin by digest:** a 21-minor-version jump on an automation engine is exactly where you want an immutable reference instead of a moving tag.

**Gotcha — cold backup of an n8n volume:** from the host, `/var/lib/docker/volumes/<vol>/_data` can look empty; the host-side view of the volume is not representative. Do cold backups from a container that mounts the volume:

```bash
docker run --rm -v <vol>:/d:ro -v "$BACKUP_DIR":/out alpine \
  tar czf /out/n8n-volume-$(date +%F).tgz -C /d .
```

**Behavior change:** since n8n 2.0 the *Execute Command* node is disabled by default. If workflows depend on it, re-enable deliberately — do not assume old workflows carry the flag forward.

### Home Assistant 2026.5.4 → 2026.10.2

Old image retagged for rollback, config backed up without logs. Side effect worth knowing: a `kasa.smart.smartdevice` log storm (~300k errors in 6 days) disappeared entirely on the new version — sometimes the "risky" major jump is also the fix for chronic noise.

## General Gotchas

- **Rebuilding a TypeScript project for a dependency bump can ship uncommitted work.** If `dist/` loads dependencies at runtime, a dependency bump needs only a service *restart* plus a type check (`tsc --noEmit`) — rebuilding from `src/` risks promoting WIP changes that were never compiled.
- **Never inspect secrets with `cat` in test scripts.** Use `test -r <file>` or `stat` to assert readability; printing contents into test output is how secrets leak into logs.
- **Prefer image retag + config backup over full snapshots** for single-service updates; reserve full snapshots for host-level changes.

## Related

- [Weekly Security Scan](security-weekly-scan.md) — the scan that decides *what* needs updating and who owns the fix.
- [Agent Media Playback](agent-media-playback.md) — same mini PC, media stack.
