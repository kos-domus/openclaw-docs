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
