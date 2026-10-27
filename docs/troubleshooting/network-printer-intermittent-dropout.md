---
title: "Network Printer Intermittently Unreachable and Silently Canceled Jobs"
slug: network-printer-intermittent-dropout
category: troubleshooting
tags: ["cups", "printing", "mesh-network", "deco", "diagnostics"]
sources:
- sessions/2026-10-26-minipc-ops-printer-youtube-docker-security-scan.md
- sessions/2026-10-27-deco-backhaul-printer-audio-memory-plan.md
last_updated: "2026-10-27"
version: 1
---

# Network Printer Intermittently Unreachable and Silently Canceled Jobs

Two failure modes that look unrelated but shared one root cause on a home mini PC server: queued print jobs disappearing, and a Wi-Fi printer dropping off the network at random intervals.

## Symptom 1: Jobs Canceled "By User" Exactly 3 Hours After Creation

**Exact state observed:** queued jobs show `canceled / job-canceled-by-user` precisely 3 hours (10800 s) after creation when the printer is unreachable.

**Cause:** CUPS cancels uncompletable jobs after `MaxJobTime` seconds. The default is 10800 s, and the cancel is recorded as user-initiated — nothing in the logs points at the timeout.

**Fix:** raise the ceiling in `cupsd.conf`:

```
MaxJobTime 604800
```

Then restart CUPS and verify with a held job: the job's `job-cancel-after` attribute must read `604800`. A 7-day ceiling keeps jobs alive across printer outages instead of silently discarding them.

> This changes *when* jobs die, not *whether the printer is reachable*. Treat it as keeping evidence, not fixing the network.

## Symptom 2: Printer (and Other Clients of One Room) Dropping Intermittently

A Wi-Fi printer shared a room with a mesh node (TP-Link Deco) that was *believed* to be Ethernet-backhauled — the cable physically arrived at the node. The printer was nonetheless unreachable at random intervals.

**Root cause:** the Ethernet cable was badly crimped (connector boot too thick, poor crimp). A Deco node that cannot establish wired uplink **does not raise an error — it silently falls back to wireless backhaul**. The weak wireless link then degrades every client attached to that node, which surfaces as intermittent dropouts of the printer (and anything else nearby).

**Key insight:** backhaul type is a *fact to read*, not an assumption. "The cable is plugged in" proves nothing.

### Diagnosing Backhaul Type via the Deco Local API

Query the mesh's node list with the Python library `tplinkrouterc6u` (authentication uses the TP-Link account password). Each node reports its uplink type toward the master — in this case the garage node reported **wireless** despite the Ethernet cable being physically connected.

> The session records the method (library + account login + per-node link type) without inline code; check the library's current README for the exact client construction, as its API has changed between versions.

If a node that should be wired reports wireless, inspect the physical link: re-crimp or replace the cable. After the cable was redone, the node came up on 1 Gbit Ethernet and the printer returned to stable IPP reachability.

## Monitoring: Catch the Pattern, Not Just the Moment

A state-change monitor (pycups + `ipptool`, systemd timer every 2 minutes) sends Telegram alerts when the printer status changes:

- offline **with pending jobs** (the dangerous combination),
- out of paper, jam, toner low.

Since the second session it also appends every reachability transition to a history log:

```
~/.local/state/stampante-history.log
```

A week of history turns "it drops sometimes" into a schedulable diagnosis — and answers follow-up questions like "is a dedicated USB cable still needed?" with data instead of guesses.

## Decision Lesson

Before buying hardware to work around a flaky device (e.g. a USB cable to bypass network printing), verify the network path of the *nearest infrastructure node*. A silent backhaul fallback one hop away reproduces exactly the symptoms that make the device itself look broken.

## Related

- [Hardware and OS Setup](../getting-started/hardware-and-os-setup.md) — the mini PC hosting CUPS.
- [CUPS documentation — `MaxJobTime`](https://www.cups.org/doc/man-cupsd.conf.html) (upstream reference).
