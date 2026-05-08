## 2026-05-08 - SKIP docs merge, tracked upstream 2026.5.7

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs were updated and no session status flips were needed.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.5.2`
- Checked upstream latest GitHub/npm version: `2026.5.7` (published 2026-05-07)
- Refreshed `docs/meta/upstream-version.yaml` verification timestamp and `docs/index.yaml` catalog timestamp for this run
- Local install remains behind upstream: local `2026.5.2` vs upstream `2026.5.7`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-08-v2026.5.7.md`

### Relevant upstream notes
- `2026.5.7`: useful operator-facing cleanup rather than fluff — cron JSON now exposes computed status, `channels list` is less noisy and more explicit, `doctor --fix` keeps/repairs Codex OAuth routing, long-lived session skill snapshots reset correctly, and there are real correctness fixes across Telegram, Discord, WhatsApp, approvals, and delivery accounting.

### Self-assessment
- Clean SKIP again: no fake extraction, just real repo maintenance.
- Worth tracking separately from `2026.5.6`: this one is not a headline release, but it quietly improves several surfaces we actually use for fleet ops and diagnostics.


## 2026-05-07 - SKIP docs merge, tracked upstream 2026.5.5 and 2026.5.6

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs were updated and no session status flips were needed.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.5.2`
- Checked upstream latest GitHub/npm version: `2026.5.6` (both `2026.5.5` and `2026.5.6` were published on 2026-05-06)
- Refreshed `docs/meta/upstream-version.yaml` verification timestamp and `docs/index.yaml` catalog timestamp for this run
- Local install remains behind upstream: local `2026.5.2` vs upstream `2026.5.6`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-07-v2026.5.5.md`
- `docs/meta/upstream-updates/2026-05-07-v2026.5.6.md`

### Relevant upstream notes
- `2026.5.5`: broad stability pass across Telegram/Discord/Matrix/Slack, session/TUI recovery, provider compatibility, and generated-media delivery — but it also introduced a risky `doctor --fix` rewrite from valid `openai-codex/*` OAuth routes to `openai/*`.
- `2026.5.6`: immediate hotfix that reverts that Codex/OpenAI routing regression and cleans up plugin/fetch timeout handling, so this is the safer upstream floor if we decide to move past `2026.5.2`.

### Self-assessment
- Clean SKIP again: no fake doc extraction, just the maintenance that mattered.
- Important catch today: `2026.5.6` is not a random patch — it is the guardrail release for anyone who might touch `doctor --fix` in the new line, so logging both `2026.5.5` and `2026.5.6` separately was worth it.


## 2026-05-06 - SKIP docs merge, tracked upstream 2026.5.4

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs were updated and no session status flips were needed.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.5.2`
- Checked upstream latest GitHub/npm version: `2026.5.4` (published 2026-05-05)
- Refreshed `docs/meta/upstream-version.yaml` verification timestamp and `docs/index.yaml` catalog timestamp for this run
- Local install remains behind upstream: local `2026.5.2` vs upstream `2026.5.4`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-06-v2026.5.4.md`

### Relevant upstream notes
- `2026.5.4`: another big operator-facing drop — Google Meet/Twilio voice path got much better, plugin install/update repair kept hardening for the npm-first plugin world, gateway/plugin hot paths got leaner, and a huge pile of fixes landed across Telegram, WhatsApp, Discord, Slack, Codex/OpenAI, cron/session handling, approvals, and Windows/macOS service edges.
- Biggest practical upgrade angle for us: it looks safer and smoother for plugin lifecycle + messaging reliability, but it is still big enough that it deserves smoke tests instead of a blind bump.

### Self-assessment
- Clean SKIP run again: no fake doc work, just real repo maintenance.
- Worth noting: the tracker already knew about `2026.5.4`, but the changelog and upstream-notes folder did not. Glad I closed that bookkeeping gap before it turns into silent drift.


## 2026-05-05 - SKIP docs merge, tracked upstream 2026.5.3 and 2026.5.3-1

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs were updated and no session status flips were needed.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.5.2`
- Checked upstream latest GitHub/npm version: `2026.5.3-1` (published 2026-05-04)
- Confirmed the intermediate stable release `2026.5.3` also landed since the last check
- Refreshed `docs/meta/upstream-version.yaml` to `last_check: 2026-05-05` / `checked_at: 2026-05-05`
- Local install is now behind upstream by the newly released `2026.5.3` line and its npm hotfix `2026.5.3-1`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-05-v2026.5.3.md`
- `docs/meta/upstream-updates/2026-05-05-v2026.5.3-1.md`

### Relevant upstream notes
- `2026.5.3`: major plugin lifecycle hardening for the npm-first world, gateway lazy-load/perf work, new file-transfer plugin/tools, shared progress streaming mode, `/steer`, and a ton of channel/runtime fixes. Biggest operator-impactful change: invalid config now fails closed instead of being auto-restored at gateway startup.
- `2026.5.3-1`: small but useful hotfix that stops false-positive install-scanner blocks on official bundled plugin packages.

### Self-assessment
- Clean SKIP run: no forced doc merge, just the maintenance that actually mattered.
- Important catch today: `2026.5.3` is not just a routine patch — the fail-closed config behavior and plugin lifecycle churn make it worth a deliberate upgrade smoke test instead of a blind bump.


## 2026-05-04 - Processed May 3 session + upstream/local install aligned on 2026.5.2

### Processed
- `sessions/2026-05-03-spesabot-orcharch-p1-spesify-ui-overhaul.md` — flipped `status: ready` → `processed` and extracted only the durable OpenClaw-adjacent automation lessons instead of pulling Spesify product implementation into the core docs set.

### Docs impact
- `docs/guides/cron-jobs-and-automation.md` (v8 → v9) — added the delivery-safe failure-monitor pattern: persist the "already notified" marker only after the outbound alert succeeds, clear it on recovery, and pre-seed state when deploying onto an already-failed service.
- `docs/reference/cron-scheduling-reference.md` (v2 → v3) — added the reference version of the same watchdog-state rule for systemd timer / cron-adjacent monitors.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.5.2`
- Checked upstream latest release with GitHub and npm: `2026.5.2` (released 2026-05-02)
- Refreshed `docs/meta/upstream-version.yaml` to `last_check: 2026-05-04` / `checked_at: 2026-05-04`
- Local install is now aligned with upstream. The tracker had been stale on `2026.4.29`, so this run records the first verified aligned state on `2026.5.2`.
- No newer stable OpenClaw release landed since the previous check, so no new `docs/meta/upstream-updates/` note was needed.

### Self-assessment
- Good restrained run: the source session was mostly downstream Spesify work, but the watchdog notification-state pattern was worth rescuing because it generalizes cleanly to OpenClaw operators.
- Glad we caught the quoted `status: "ready"` session instead of falsely calling this a SKIP.


## 2026-05-03 - Processed May 1-2 sessions + tracked upstream 2026.5.2

### Processed
- `sessions/2026-05-01-spesabot-image-library-and-mc-todo-fixes.md` — flipped `status: ready` → `processed` and extracted the reusable operator lessons instead of the SpesaBot-specific implementation details.
- `sessions/2026-05-02-tools-wikilinks-orvea-toast-cleanup.md` — flipped `status: ready` → `processed` and promoted the durable OpenClaw patterns around MCP scope, secret handling, cron maintenance, and localhost-only service access.

### Docs impact
- `docs/guides/skills-and-slash-commands.md` (v2 → v3) — documented the durable Claude Code slash-command pattern: user-scope vs project-local command files, frontmatter metadata, and `$ARGUMENTS` substitution.
- `docs/reference/mcp-servers.md` (v1 → v2) — added MCP scope guidance (user vs project) and the wrapper-based secret-injection pattern for paid MCP services such as Firecrawl.
- `docs/guides/cron-jobs-and-automation.md` (v7 → v8) — added two operator rules: patch recurring jobs in place with `openclaw cron edit`, and treat the first live scheduled run as part of the implementation rather than an afterthought.
- `docs/guides/remote-access.md` (v2 → v3) — added the SSH-over-Tailscale localhost tunnel pattern plus keepalive settings for long-lived maintenance sessions.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.4.29`
- Checked upstream latest release with GitHub and npm: `2026.5.2` (released 2026-05-02)
- Refreshed `docs/meta/upstream-version.yaml` to `last_check: 2026-05-03` / `checked_at: 2026-05-03`
- Local install is behind upstream by one stable release: `2026.5.2`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-03-v2026.5.2.md`

### Relevant upstream notes
- `2026.5.2`: plugin management now understands the npm-first cutover better, gateway/agent hot paths are leaner, WebChat/Control UI got another resilience pass, messaging fixes landed across Telegram/WhatsApp/Discord/Slack/Signal, and provider-media fixes touched Firecrawl/web search plus OpenAI-compatible TTS/Realtime paths.

### Self-assessment
- Good cut today: the source sessions were heavy on downstream product work, but there were still four real operator patterns worth rescuing into the docs.
- I like that I kept the repo focused on reusable OpenClaw knowledge instead of bloating it with SpesaBot details that do not generalize.
- Follow-up worth considering: the MCP reference is now materially more practical, but it still lacks a broader section on local-vs-remote MCP deployment tradeoffs.

## 2026-05-01 - Upgraded local OpenClaw to 2026.4.29

### Upgrade
- Local install: **2026.4.24 → 2026.4.29** (single-hop, four stable releases skipped)
- Pre-flight smoke test: confirmed the breaking change "restrictive profiles
  `minimal`/`messaging` no longer inherit `tools.exec`/`tools.fs`" does NOT
  impact this fleet. 6/8 agents (orchestrator, cso, frontend-specialist,
  backend-expert, orchestration-architect, wip) use the `coding` profile,
  which retains fs+exec. The other 2 (cos, family) have explicit
  `tools.allow` arrays that bypass profile inheritance entirely. Zero
  agents on `minimal` or `messaging`.
- Procedure: backup `openclaw.json` → stop gateway → `npm install -g openclaw@2026.4.29`
  → start gateway → smoke test.
- Post-upgrade state: gateway active, 8/8 agents registered, 51/51 cron
  jobs preserved, 9/9 routing bindings intact, zero errors in journal.
- `docs/meta/upstream-version.yaml`: bumped `installed.local_cli_version`
  to 2026.4.29 + recorded `upgraded_from`/`upgraded_at` for trace.
- Backup retained at `~/.openclaw/openclaw.json.before-2026.4.29`.

## 2026-05-01 - Processed fleet/Telegram docs sessions + tracked upstream 2026.4.29

### Processed
- `sessions/2026-04-30-fleet-fixes-spesabot-consolidation-esselunga-image-registry.md` — flipped `status: ready` → `processed` and extracted durable OpenClaw lessons into the docs.
- `sessions/2026-04-29-sacchitalia-ddt-pipeline-and-backend-mvp.md` — flipped `status: ready` → `processed` after review, but did **not** promote core docs because the reusable value was mostly downstream-product-specific rather than OpenClaw-specific.
- `sessions/2026-04-28-telegram-capture-and-fleet-routing-overhaul.md` — flipped `status: ready` → `processed` and extracted the durable Telegram topic-routing lessons into the docs.

### Docs impact
- `docs/guides/cron-jobs-and-automation.md` (v6 → v7) — added two durable patterns: staggering dependent morning jobs and using stdout redirection instead of `gws drive files get -o` for text responses in unattended cron wrappers.
- `docs/reference/cron-scheduling-reference.md` (v1 → v2) — added dependency-aware scheduling and the "bootstrap empty state outside the main job" pattern for noisy-but-optional files like daily memory.
- `docs/reference/gws-cli-reference.md` (v1 → v2) — documented the binary-only reality of `gws drive files get -o <path>`.
- `docs/guides/telegram-bot.md` (v3 → v5) — documented both the `delivery.accountId` / bot-membership trap behind misleading Telegram `chat not found` errors and the current direct-config pattern for topic-scoped Telegram bindings.
- `docs/troubleshooting/common-errors.md` (v8 → v10) — added concrete error entries for GWS text downloads, Telegram wrong-account delivery, recurring daily-memory `ENOENT` noise, CLI bind over-broadness on topics, and `SOUL.md` bootstrap truncation.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.4.24`
- Checked official upstream latest release: `2026.4.29` (GitHub release published 2026-04-30; npm tracker also on `2026.4.29`)
- Refreshed `docs/meta/upstream-version.yaml` to `last_check: 2026-05-01` / `checked_at: 2026-05-01`
- Local install is behind upstream by four stable releases: `2026.4.25`, `2026.4.26`, `2026.4.27`, and `2026.4.29`
- There is **no** stable `2026.4.28` release tag to track, so the gap is real even though the numbering skips

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-05-01-v2026.4.29.md`

### Relevant upstream notes
- `2026.4.29`: active-run steering defaults, people-aware memory/wiki features, NVIDIA provider onboarding, richer startup diagnostics, broad channel reliability fixes, and a key operator-impactful change where restrictive profiles no longer implicitly inherit `tools.exec` or `tools.fs`.

### Self-assessment
- Good extraction today: the Apr 30 and Apr 28 sessions had real reusable OpenClaw knowledge, and I promoted only the durable ops patterns instead of dragging product-specific details into this repo.
- Also glad I did **not** force the Sacchitalia session into core docs. It had useful engineering work, but almost all of it belongs to the downstream product, not the OpenClaw knowledge base.
- Main follow-up: before upgrading local OpenClaw, smoke-test every agent that uses a restrictive tool profile, because `2026.4.29` removes the old implicit `exec` / `fs` widening.

## 2026-04-30 - SKIP docs merge, tracked upstream 2026.4.27

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diataxis docs were updated and no session status flips were needed.
- Confirmed the untracked `sessions/2026-04-28-excalidraw-toolkit-and-spec-dsl.md` is already marked `status: processed`, so it did not belong to today's elaboration queue.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.4.24`
- Checked upstream latest release with GitHub and npm: `2026.4.27` (released 2026-04-29)
- Confirmed `docs/meta/upstream-version.yaml` already tracks `last_check: 2026-04-30`, upstream `2026.4.27`, and local `2026.4.24`
- Local install is behind upstream by one tracked stable release: `2026.4.27`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-04-30-v2026.4.27.md`

### Relevant upstream notes
- `2026.4.27`: Codex Computer Use setup with `/codex computer-use status/install`, DeepInfra bundled provider, Tencent Yuanbao and QQBot channel expansions, plugin manifest-first model catalogs (reducing Gateway boot work), operator-managed outbound proxy routing, broad Plugin SDK testing migration to focused subpaths, and fixes across device pairing, Telegram multi-bot approvals, Slack socket/media stalls, gateway startup prewarm, auto-reply drain timeouts, and CLI update/status reliability.

### Self-assessment
- Clean maintenance run. The only valuable work was capturing the 2026.4.27 release notes, which is a significant release with new providers, channels, and plugin SDK restructuring.
- The local CLI gap (2026.4.24 vs 2026.4.27) is narrower than past runs but still worth tracking for upgrade planning.

## 2026-04-28 - SKIP docs merge, tracked upstream 2026.4.25 and 2026.4.26

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs were updated and no session status flips were needed.
- Confirmed the newly present `sessions/2026-04-27-fleet-multi-agent-tasks-obsidian-vault-mcp-integration.md` is already marked `status: processed`, so it did not belong to today's elaboration queue.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.4.24`
- Checked upstream latest release with GitHub and npm: `2026.4.26`
- Confirmed `docs/meta/upstream-version.yaml` now tracks `last_check: 2026-04-28`, upstream `2026.4.26`, and local `2026.4.24`
- Local install is behind upstream by two stable releases: `2026.4.25` and `2026.4.26`

### New upstream release notes captured
- `docs/meta/upstream-updates/2026-04-28-v2026.4.25.md`
- `docs/meta/upstream-updates/2026-04-28-v2026.4.26.md`

### Relevant upstream notes
- `2026.4.25`: major TTS expansion, persisted plugin-registry/cold-start cleanup, broader OpenTelemetry coverage, browser/control-ui/install hardening, and fixes across WhatsApp, Bonjour, ACP/Codex, hidden runtime context handling, and package update reliability.
- `2026.4.26`: Google Live browser Talk transport, Cerebras provider, new `openclaw migrate` import flows, better raw config diffing, and fixes for `sessions_spawn` model overrides, explicit ACP spawns, cron model fail-closed behavior, WhatsApp proxy-aware QR login, and Bonjour hostname defaults.

### Self-assessment
- Clean restraint today: no forced doc merge just to make the run feel busy.
- The valuable work was upstream bookkeeping — the repo now captures both skipped stable releases instead of silently jumping from 4.24 to 4.26.
- Main follow-up: plan the local upgrade from `2026.4.24` to `2026.4.26`, especially for WhatsApp, Bonjour, cron model enforcement, and subagent model-override fixes.

## 2026-04-27 - Processed Saas Delivery Receipts session + upstream aligned

### Processed
- `sessions/2026-04-25-saas-delivery-receipts-prototype-and-stack-pivot.md` — flipped `status: ready` → `processed` after review.

### Docs impact
- No core OpenClaw docs were updated from this session. The session is valuable, but its durable content is downstream-project-specific (Sacchitalia / ESOLVER / JSAPP architecture, extraction harness milestones, client-side stack tradeoffs) rather than reusable OpenClaw knowledge.
- Confirmed that the AJV `NodeNext`/ESM interop note is technically useful but still too generic for this repository unless the same pattern starts showing up across OpenClaw-facing docs or sessions.

### Upstream consistency check
- Checked installed CLI version with `openclaw --version`: `2026.4.24`
- Checked upstream latest release with GitHub and npm: `2026.4.24`
- Refreshed `docs/meta/upstream-version.yaml` to `last_check: 2026-04-27` and `checked_at: 2026-04-27`
- No new upstream stable releases landed since the previous check, so no new `docs/meta/upstream-updates/` note was needed

### Housekeeping
- Refreshed `docs/index.yaml` metadata timestamp so the catalog reflects the current elaboration run

### Self-assessment
- Good restraint here: I processed the ready session and kept the docs repo clean instead of forcing project-specific material into OpenClaw documentation just to "extract something".
- The repo is now consistent again: no lingering `ready` session from Apr 25, upstream tracking is current, and yesterday's 4.24 docs changes remain ready to ship.
- Follow-up worth considering later: define a formal policy for `processed-without-doc-changes` sessions so this edge case is explicit rather than implied.

## 2026-04-26 - Cross-provider fallback rule + 4.24 upstream

### Updated
- `reference/agent-fleet-reference.md` (v2 → v3) — added "Cross-provider fallback rule" section with three-tier template (critical / standard / background); added "OpenAI Codex auth fragility" subsection; flagged the Apr 25 Gemini saturation incident; expanded tags with `fallback`, `providers`.
- `troubleshooting/common-errors.md` (v6 → v8) — new "Channels and Providers" section opening with a "Diagnostic principle: check the gateway before the channel" subsection (recipe + real example from the bonjour-induced WhatsApp 499 incident), followed by entries for WhatsApp 499 reconnect loop, Codex `refresh_token_reused` / `credential unavailable`, Gemini `All models failed` saturation. Each entry has diagnostic checks, recovery procedure, and cross-links to the agent fleet doc.

### Added
- `meta/upstream-updates/2026-04-26-v2026.4.24.md` — captures the local install gap (4.15 → 4.24), security floor at 4.23, breaking change on plugin SDK / tool-result middleware, smoke-test commands, and back-references to the three open ops issues.

### Sources
- `memory/reports/openclaw-monitor/2026-04-26.md` (Kos daily ops report — Apr 25 wrap-up + Apr 26 upstream scan)

### Self-assessment
- Doc cross-linking (fleet ↔ troubleshooting ↔ upstream) tightened: each operational lesson has exactly one canonical home with back-links from related docs.
- "All models failed" promoted from implicit fleet-doc lesson to an explicit error entry so future search hits land on copy-pasteable recovery instead of discussion.
- Open question: should Kos monitor reports get their own session-class folder under `sessions/` or stay as `memory/reports/...` references? Currently cited as sources but outside the session-validation pipeline.

## 2026-04-25 - OpenClaw 2026.4.23 Update

### OpenClaw Core Update (v2026.4.23)

Nuove versioni di OpenClaw (fino alla 2026.4.23) sono state rilasciate, ma la versione locale installata è ancora la 2026.4.15.

#### Breaking Changes:
- **Models**: Default model per OpenAI image generation aggiornato a `gpt-image-2`.
- **Cron**: Cron runtime state separato in `jobs-state.json`.

#### New Features:
- Migliorato l'onboarding/setup wizard.
- Costi stimati migliorati e supporto per Moonshot Kimi.
- I nested agent lanes sono ora limitati per target session.

#### Fixes:
- **Security**: Fissata vulnerabilità (GHSA-c28g-vh7m-fm7v) sui comandi owner-enforced, bloccato `NODE_OPTIONS` per MCP stdio.
- **Dependencies**: `openclaw doctor` ora ripara meglio le dipendenze runtime dei plugin.
- **Stability**: Pruning aggressivo del session store per prevenire OOM errors. Fixes per plugin Anthropic, cron delivery, e Telegram callbacks.

---

### Self-assessment:
Ho verificato la presenza di sessioni `ready` in `sessions/` ma non ne ho trovate (SKIP docs generation). Ho controllato la consistenza upstream rilevando un disallineamento: la versione installata è 2026.4.15 mentre l'ultima release upstream è 2026.4.23. Ho aggiornato `upstream-version.yaml` di conseguenza, inserito le note di rilascio riassuntive nel `CHANGELOG.md` e committato.

---
date: 2026-04-10
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs required content merges today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub and npm: `2026.4.9`
- Confirmed `docs/meta/upstream-version.yaml` now tracks upstream `2026.4.9` with `last_check: 2026-04-10`
- Recorded upstream summary in `docs/meta/upstream-updates/2026-04-10-v2026.4.9.md`

### Relevant upstream notes for 2026.4.9

- New grounded memory/dreaming backfill and improved diary tooling
- Better provider auth aliasing and auth-profile persistence
- Stronger browser and `.env` security hardening
- Safer remote node exec event sanitization
- Better cron and agent timeout behavior
- Session routing and reply-token leakage fixes across external channels

### Self-assessment

Run completed cleanly. No session backlog existed, upstream was checked properly, and the repo was updated with the new upstream summary plus version-tracker refresh. Main follow-up: consider upgrading the local OpenClaw install from `2026.4.8` to `2026.4.9`.

---
date: 2026-04-11
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs required content merges today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub and npm: `2026.4.10`
- Confirmed `docs/meta/upstream-version.yaml` now tracks upstream `2026.4.10` with `last_check: 2026-04-11`
- Recorded upstream summary in `docs/meta/upstream-updates/2026-04-11-v2026.4.10.md`
- Local install is still behind upstream by two releases (`2026.4.9` and `2026.4.10`)

### Relevant upstream notes for 2026.4.9 and 2026.4.10

- `2026.4.9` added grounded memory and diary improvements, better auth aliasing, stronger browser and `.env` hardening, safer remote exec event handling, and better cron/session routing behavior
- `2026.4.10` added the bundled Codex provider path, the optional Active Memory plugin, `openclaw exec-policy`, remote `commands.list`, and experimental MLX speech support on macOS
- `2026.4.10` also ships broad security hardening across browser, exec, plugins, gateway startup, cron reliability, and multi-channel thread routing

### Self-assessment

Run completed cleanly. There was no session backlog to elaborate, upstream tracking is now current, and the changelog captures the newly missed upstream release plus the local version gap. Main follow-up: upgrade the local OpenClaw install from `2026.4.8` to at least `2026.4.10` and smoke-test ACP/Codex, cron, and browser-related flows.

---
date: 2026-04-12
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs required content merges today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub and npm: `2026.4.11`
- `docs/meta/upstream-version.yaml` already tracks upstream `2026.4.11` with `last_check: 2026-04-12`
- Recorded upstream summary in `docs/meta/upstream-updates/2026-04-12-v2026.4.11.md`
- Local install is still behind upstream by three releases (`2026.4.9`, `2026.4.10`, and `2026.4.11`)

### Relevant upstream notes for 2026.4.11

- Dreaming gained ChatGPT import ingestion plus new `Imported Insights` and `Memory Palace` diary subtabs
- Webchat/control UI now supports structured assistant media/reply/voice bubbles and gated `[embed ...]` rich output
- `video_generate` added richer provider options, adaptive aspect ratios, reference audio inputs, and URL-only asset delivery
- Plugin manifests can now declare activation/setup descriptors for auth, pairing, and config flows
- ACP parent streams now suppress leaked child commentary, and timeout handling is more consistent
- Channel fixes landed for Telegram topic routing, WhatsApp reactions/default-account handling, and Google Veo request compatibility

### Self-assessment

Run completed cleanly. No ready-session backlog existed, upstream tracking was already on the newest known version, and this run filled the missing local changelog/update-summary entry for `2026.4.11`. Main follow-up: upgrade local OpenClaw from `2026.4.8` to at least `2026.4.11` and smoke-test ACP, Telegram topic sessions, WhatsApp reactions, and packaged CLI completion generation.

---
date: 2026-04-13
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs required content merges today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub: `2026.4.11`
- Confirmed `docs/meta/upstream-version.yaml` remains aligned to upstream `2026.4.11` and refreshed `last_check: 2026-04-13`
- Local install is still behind upstream by three releases (`2026.4.9`, `2026.4.10`, and `2026.4.11`)

### Self-assessment

Run completed cleanly. No ready sessions needed elaboration, upstream tracking is still aligned, and today only required a tracker refresh plus the daily audit trail entry. Main follow-up: upgrade the local OpenClaw install from `2026.4.8` to at least `2026.4.11`.

---
date: 2026-04-14
type: documentation-run
---
### Daily knowledge base run

Processed nine session files that were marked `status: ready` and promoted their reusable content into new docs.

### Docs created

- `docs/guides/deep-research-pipeline.md`
- `docs/reference/deep-research-reference.md`
- `docs/reference/environment-variables.md`
- `docs/reference/agent-fleet-reference.md`
- `docs/reference/cron-scheduling-reference.md`
- `docs/reference/1password-integration-reference.md`
- `docs/troubleshooting/deep-research-pipeline-issues.md`

### Session processing

- Flipped all processed session files from `ready` to `processed`
- Updated `docs/index.yaml` metadata and catalog counts
- Focused extraction on reusable OpenClaw knowledge, especially deep-research architecture, env loading, secret caching, fleet configuration, and scheduling patterns
- Left project-specific SpesaBot details inside session history when they were product-specific rather than core OpenClaw docs material

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Tracker file `docs/meta/upstream-version.yaml` refreshed with `last_check: 2026-04-14`
- Tracker still points to upstream latest stable `2026.4.12`
- Local install is behind upstream by four stable releases: `2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`
- Added missing upstream summary file: `docs/meta/upstream-updates/2026-04-13-v2026.4.12.md`
- Also noted that `2026.4.14-beta.1` exists upstream as a prerelease, but this repo continues to track stable releases in the version tracker

### Relevant upstream notes for 2026.4.12

- New optional Active Memory plugin with a dedicated memory sub-agent
- Experimental MLX speech provider for local Talk Mode on macOS
- New `openclaw exec-policy` command family for syncing requested exec config with local approvals
- New `commands.list` RPC for runtime command discovery
- More plugin, memory, dreaming, and setup reliability work

### Self-assessment

This run was solid. The repo had real session backlog, and I converted the reusable parts into proper reference, guide, and troubleshooting docs instead of dumping project logs into the docs tree. The weak spot is that SpesaBot-specific implementation details were intentionally not expanded into standalone docs because they are downstream product work, not core OpenClaw documentation.

---
date: 2026-04-15
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis docs required content merges today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub and npm: `2026.4.14`
- Confirmed `docs/meta/upstream-version.yaml` now tracks upstream `2026.4.14` with `last_check: 2026-04-15`
- Recorded upstream summary in `docs/meta/upstream-updates/2026-04-15-v2026.4.14.md`
- Local install is still behind upstream by five stable releases (`2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`, and `2026.4.14`; no `2026.4.13` stable release was published)

### Relevant upstream notes for 2026.4.14

- GPT-5.4 and Codex compatibility improved, including forward-compat support for `gpt-5.4-pro`
- Telegram forum topics now preserve human-readable topic names more consistently
- Security hardening landed for gateway config patching, browser SSRF enforcement, allowlists, and attachment path checks
- Reliability fixes landed across subagents, cron, browser CDP, session routing, media handling, and provider/proxy integrations

### Self-assessment

Clean run. No ready-session backlog today, but the upstream audit mattered because the tracker had moved to `2026.4.14` and the repo was missing the release summary entry. Biggest follow-up is still obvious: local OpenClaw is lagging hard on `2026.4.8`, so an upgrade plus smoke tests for subagents, Telegram topics, cron, browser, and provider flows would be worth doing soon.


---
date: 2026-04-15
type: upstream-check
---
### Daily knowledge base rerun

No session files with `status: ready` were found under `sessions/`, so there was no Diátaxis content to merge and no session status flips were needed.

### Upstream consistency check

- Re-checked installed CLI version with `openclaw --version`: `2026.4.8`
- Re-checked upstream latest release with GitHub and npm: `2026.4.14`
- Confirmed `docs/meta/upstream-version.yaml` is still aligned to upstream `2026.4.14`
- Confirmed there is still no `2026.4.13` stable release, so the tracked stable chain remains correct
- Local install remains behind upstream by five stable releases: `2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`, and `2026.4.14`

### Relevant upstream notes

- No new upstream release landed since the last check on `2026-04-15`
- The current latest stable remains `2026.4.14`, with the same headline themes: GPT-5.4/Codex compatibility improvements, Telegram topic-name handling, broad security hardening, and reliability fixes across subagents, cron, browser, routing, and provider integrations

### Self-assessment

This was a clean no-op rerun. No session backlog existed, upstream tracking was already current, and I verified that the repo did not miss a newer stable release after the earlier `2026-04-15` audit. The only real follow-up is still the same one: upgrade the local OpenClaw install from `2026.4.8` when convenient.


---
date: 2026-04-16
type: upstream-check
---
### Daily knowledge base run

No session files with `status: ready` were found under `sessions/`, so no Diátaxis content merges or session status flips were needed today.

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub and npm: `2026.4.14`
- Confirmed `docs/meta/upstream-version.yaml` remains aligned to upstream `2026.4.14` and refreshed `last_check: 2026-04-16`
- Confirmed there is still no newer stable release beyond `2026.4.14`, and no missing intermediate stable versions after the already tracked gap
- Local install remains behind upstream by five stable releases: `2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`, and `2026.4.14`

### Relevant upstream notes

- No new OpenClaw stable release landed since the previous check
- The current latest stable is still `2026.4.14`, with the same headline themes: GPT-5.4/Codex compatibility improvements, Telegram forum topic-name persistence, SSRF and config hardening, and broad reliability fixes across subagents, cron, browser/CDP, routing, and media handling

### Self-assessment

Clean maintenance run. There was no docs backlog to elaborate, the upstream tracker is still correct, and the repo already had the needed release summary for `2026.4.14`. The only real follow-up is still upgrading the local OpenClaw install from `2026.4.8`.

---
date: 2026-04-17
type: documentation-run
---
### Daily knowledge base run

Processed three session files that were still marked `status: ready` and merged only the reusable, OpenClaw-adjacent patterns into the docs set.

### Docs updated

- **Updated**: `deep-research-pipeline` — added guidance for browser-backed discovery, persistent-profile flows, enrichment APIs, text-layer versus vision decisions, and campaign-safe aggregation
- **Updated**: `deep-research-reference` — added reference patterns for persistent browser profiles, secondary enrichment sources, batching, dedupe keys, and browser-session API discovery
- **Updated**: `deep-research-pipeline-issues` — documented selector drift, regional asset filtering, campaign wipeouts from partial ingest, optional enrichment, and non-interactive env failures
- **Updated**: `environment-variables` — added the stable `op-env-cached.sh` sourcing pattern for non-interactive services
- **Sources**: `sessions/2026-04-15-conad-flyer-parser-and-discovery.md`, `sessions/2026-04-15-conad-flyer-parser-and-discovery-2.md`, `sessions/2026-04-16-spesify-pipeline-images-matching-ui.md`

### Session processing

- Flipped all three processed session files from `ready` to `processed`
- Refreshed `docs/index.yaml` metadata and source links
- Deliberately did **not** create downstream product docs for SpesaBot-specific business logic, because that would pollute the OpenClaw knowledge base with app-specific details

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream latest release with GitHub releases and npm: `2026.4.15`
- Confirmed `docs/meta/upstream-version.yaml` now tracks upstream `2026.4.15` with `last_check: 2026-04-17`
- Recorded upstream summary in `docs/meta/upstream-updates/2026-04-17-v2026.4.15.md`
- Local install remains behind upstream by six stable releases: `2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`, `2026.4.14`, and `2026.4.15`

### Relevant upstream notes for 2026.4.15

- Bundled Google TTS support, updated Anthropic defaults, and a new model-auth status card in the Control UI
- Significant security hardening around tool collisions, media embedding, file/path access, approval redaction, `memory_get`, and auth reload behavior
- Broad reliability fixes across Codex, cron announce delivery, replay recovery, Telegram documents, WhatsApp reconnects, BlueBubbles catchup, and plugin cache invalidation

### Self-assessment

This run was useful, but it needed restraint. The ready sessions were mostly downstream retail-product work, so the right move was to extract only the durable OpenClaw patterns, especially browser-backed discovery, enrichment strategy, batching, and non-interactive env loading. The weak spot is that these sessions still mix framework learnings with app-specific implementation details, which makes daily elaboration slower than it should be.



---
date: 2026-04-19
type: documentation-run
---
### Daily knowledge base run

Processed three session files that were marked `status: ready` and merged only the reusable OpenClaw-adjacent patterns into the docs set.

### Docs updated

- **Updated**: `guides/telegram-bot.md` — added Telegram Mini App companion patterns for auth reuse, WebView caching, `LocationManager`, sidebar navigation, and API security basics
- **Updated**: `troubleshooting/integration-issues.md` — added Mini App auth replay pitfalls, WebView location hangs, CSP/Helmet regressions, user-mode systemd hardening limits, and post-pipeline wrapper gotchas
- **Updated**: `guides/cron-jobs-and-automation.md` — added post-run follow-up chaining and idempotent notification-daemon patterns
- **Sources**: `sessions/2026-04-17-spesify-security-ux-stores-launch.md`, `sessions/2026-04-17-spesify-sidebar-profile-search-chaininfo-2.md`, `sessions/2026-04-18-spesify-nearby-store-drilldown-watchlist.md`

### Session processing

- Flipped all three processed session files from `ready` to `processed`
- Refreshed `docs/index.yaml` metadata and source links
- Deliberately kept product-specific Spesify business logic out of the docs set and extracted only the durable integration and automation patterns

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Checked upstream stable release line with GitHub releases and npm: `2026.4.15`
- Confirmed `docs/meta/upstream-version.yaml` still correctly tracks stable upstream `2026.4.15` and refreshed `last_check: 2026-04-19`
- Local install remains behind upstream by six stable releases: `2026.4.9`, `2026.4.10`, `2026.4.11`, `2026.4.12`, `2026.4.14`, and `2026.4.15`

### New upstream prerelease observed

- GitHub now also shows prerelease `2026.4.19-beta.1` (published `2026-04-19`)
- Headline changes in that beta: cross-agent subagent/channel-account routing fixes, safer Telegram callback watermark handling, better remote CDP diagnostics and loopback alias handling, and corrected Codex context-usage reporting
- No newer stable release than `2026.4.15` was published at the time of this run

### Self-assessment

Good run. The source sessions were mostly downstream app work, but there were still solid reusable lessons around Telegram Mini Apps, WebView quirks, cron chaining, and service hardening. The extraction stayed disciplined instead of turning the knowledge base into product notes.

---
date: 2026-04-20
type: documentation-run
---
### Daily knowledge base run

- No session files were marked `status: ready`, so no docs content needed elaboration today
- Refreshed `docs/index.yaml` metadata timestamp to reflect the maintenance run

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.8`
- Confirmed `docs/meta/upstream-version.yaml` still tracks upstream stable `2026.4.15`
- Refreshed `last_check: 2026-04-20` and recorded the currently installed local CLI version for comparison
- No newer stable release than `2026.4.15` was observed during this run
- Local install is still behind known stable upstream (`2026.4.9` through `2026.4.15`, excluding `2026.4.13` which was not part of the tracked stable line)

### Self-assessment

Clean maintenance-only run. No new source sessions meant no risk of polluting the docs set with low-signal material, and the only useful work was keeping the upstream tracking metadata honest. The weak spot is still the mismatch between installed local CLI and the tracked upstream stable line, but that belongs to upgrade ops, not docs elaboration.

---
date: 2026-04-22
type: documentation-run
---
### Daily knowledge base run

Processed two session files that were marked `status: ready`. One produced durable OpenClaw docs updates; the other was intentionally logged as product-specific and kept out of the docs tree.

### Docs updated

- **Updated**: `docs/getting-started/first-run-verification.md` — added explicit post-upgrade gateway restart verification and a quick ACP smoke-test note
- **Updated**: `docs/troubleshooting/common-errors.md` — marked the ACP stdout banner bug as fixed in `2026.4.15` and documented the `models auth login --set-default`, plugin-version, and `doctor --repair` service-overwrite gotchas
- **Updated**: `docs/reference/agent-fleet-reference.md` — added model-ownership and per-agent OAuth token lessons for multi-agent fleets
- **Source promoted**: `sessions/2026-04-20-openclaw-upgrade-4.15-gateway-restart.md`
- **Source intentionally not promoted**: `sessions/2026-04-20-spesify-conad-image-mismatch-fix.md` because it is downstream product logic, not reusable OpenClaw documentation

### Session processing

- Flipped both ready session files from `ready` to `processed`
- Refreshed `docs/index.yaml` metadata and source catalog
- Kept the docs set clean by extracting only durable framework/operator lessons

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.15`
- Checked upstream latest release with GitHub and npm: `2026.4.21`
- Updated `docs/meta/upstream-version.yaml` to track upstream `2026.4.21` with `last_check: 2026-04-22`
- Recorded upstream summaries in `docs/meta/upstream-updates/2026-04-22-v2026.4.20.md` and `docs/meta/upstream-updates/2026-04-22-v2026.4.21.md`
- Local install is behind upstream by two stable releases: `2026.4.20` and `2026.4.21`

### Relevant upstream notes for 2026.4.20 and 2026.4.21

- `2026.4.20` strengthened default prompts, pruned oversized session stores by default, split cron runtime state into `jobs-state.json`, improved pairing diagnostics, and landed a huge sweep of fixes across exec, Codex, plugins, browser, Telegram, cron, and channel reliability
- `2026.4.21` switched bundled image generation defaults to `gpt-image-2`, let `openclaw doctor` repair bundled plugin runtime dependencies, tightened owner-only command enforcement, and shipped smaller Slack/browser/npm fixes

### Self-assessment

Good run. The upgrade session had real operator-grade lessons worth preserving, especially around post-upgrade activation, ACP verification, auth-profile footguns, and multi-agent token behavior. I kept the Spesify scrape fix out of the docs on purpose, which was the right restraint.

---
date: 2026-04-24
type: documentation-run
---
### Daily knowledge base run

Processed one session file marked `status: ready` and extracted only the reusable operator lesson instead of promoting downstream app logic into the docs set.

### Docs updated

- **Updated**: `docs/troubleshooting/common-errors.md` — added a recovery pattern for accidentally overwritten files using Claude Code JSONL transcripts, plus the follow-up rule that `src/` must remain the source of truth rather than `dist/`
- **Source promoted**: `sessions/2026-04-23-spesify-major-rebuild-fixes.md`

### Session processing

- Flipped the processed session file from `ready` to `processed`
- Refreshed `docs/index.yaml` metadata and source catalog
- Kept the docs tree clean by refusing to turn Spesify-specific ingest, schema, and UI logic into fake OpenClaw documentation

### Upstream consistency check

- Checked installed CLI version with `openclaw --version`: `2026.4.15`
- Checked upstream latest release with GitHub and npm: `2026.4.22`
- Confirmed `docs/meta/upstream-version.yaml` already tracks upstream `2026.4.22` with `last_check: 2026-04-24`
- Recorded missing upstream summary in `docs/meta/upstream-updates/2026-04-24-v2026.4.22.md`
- Local install remains behind upstream (`2026.4.15` vs `2026.4.22`)

### Relevant upstream notes for 2026.4.22

- xAI gained image generation, TTS, STT, and realtime transcription support, while Deepgram, ElevenLabs, and Mistral also gained more voice-call and transcription coverage
- The TUI now supports local embedded mode without a Gateway, onboarding can auto-install missing plugins, and `/models add` can register models from chat without a restart
- Diagnostics exports, GPT-5 overlay unification, Codex/ACP work, cron and MCP cleanup, and broad channel reliability fixes make this a significant operator release
- Operator-impactful changes include the removal of Codex auth import from `~/.codex` during onboarding and the removal of the legacy `OPENCLAW_CODEX_APP_SERVER_GUARDIAN` shortcut

### Self-assessment

This was the right kind of disciplined run. The source session was mostly product work, but there was one durable recovery pattern worth preserving, and I kept the rest out of the docs instead of padding the knowledge base with irrelevant app details. The only real follow-up is operational, not editorial: the local CLI is still behind upstream by several releases.
