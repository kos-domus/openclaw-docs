# Changelog

All elaboration runs by Kos are documented here.

## 2026-04-03 — Daily Elaboration Pass (SKIP)

- **Sessions**: no session files with `status: ready` were found under `sessions/`; no doc pages required editorial updates today
- **Docs/index**: left unchanged because the corpus and source coverage did not move
- **Upstream consistency check**:
  - Installed version: `OpenClaw 2026.4.1`
  - Upstream tracked latest in `docs/meta/upstream-version.yaml`: `2026.4.2`
  - Result: local install is **behind upstream by one patch release**
- **Upstream release delta**:
  - No new release was detected beyond the already tracked `2026.4.2`, so no new upstream note file was needed today
  - Still relevant while the host remains behind: `2026.4.2` introduced Task Flow recovery/inspection primitives, managed child task spawning, plugin-owned task-flow/runtime seams, Android assistant entrypoints, plugin-owned xAI/Firecrawl config migrations, and multiple fixes around exec approvals, subagent routing, gateway pairing, and transport policy
- **Actionable discrepancy**: the docs metadata is already tracking `2026.4.2`, but this machine still runs `2026.4.1`; any behavior depending on the `2026.4.2` fixes or breaking config moves should be validated against the installed build before relying on docs that describe the newer line

### Self-Assessment
- Good no-BS pass: skipped editorial churn because there was nothing new to merge
- Upstream hygiene is acceptable: no missed release since the last check, but the host/package drift is still real and worth keeping visible
- Biggest risk remains operational rather than documentary: local runtime lag can make the docs look more current than the machine actually is


## 2026-04-02 — Daily Elaboration Pass

- **1 ready session processed**: `2026-04-01-openclaw-v31-acp-kos-pipeline-kai-mensa.md`
- **Docs updated**:
  - `docs/guides/cron-jobs-and-automation.md` now documents cron exec approval strategy, allowlist-based autonomy, and gateway restart requirements after approval changes
  - `docs/troubleshooting/common-errors.md` now includes invalid `autoApprove` config attempts, cron approval dead-ends, and the ACP JSON-stream banner bug
- **index.yaml**: refreshed timestamp, source coverage, and version counters for the updated docs
- **Session status**: flipped the processed session from `ready` to `processed`
- **Upstream consistency check**:
  - Installed version: `OpenClaw 2026.3.31`
  - Upstream tracked latest: `2026.4.2`
  - Result: local install is **behind upstream by one release line**
- **Upstream notes captured**: added `docs/meta/upstream-updates/2026-04-02-v2026.4.2.md`
- **Release delta summary added**:
  - `2026.4.1-beta.1`: approval routing, cron schema handling, pairing/config fixes
  - `2026.4.2`: `/tasks`, SearXNG web search provider, Bedrock Guardrails, cron per-job tool allowlists, improved exec approval reliability, and multiple gateway/runtime fixes relevant to automation
- **Actionable discrepancy**: repo metadata now tracks upstream `2026.4.2`, but the host still runs `2026.3.31`; upgrade should be evaluated before relying on the newer cron/approval fixes

### Self-Assessment
- Clean merge pass: this session reinforced existing automation/troubleshooting docs without needing a new page
- Upstream hygiene is better now: the repo was stale versus the official changelog, and that drift is now explicit
- Biggest unresolved gap is operational, not editorial: the docs are aligned to upstream awareness, but the installed OpenClaw version is still behind


## 2026-03-30 — Editorial Sanitization Pass

- **Sanitized**: Normalized all non-standard placeholder identifiers across the docs corpus to consistent neutral forms
- **Conventions applied**:
  - `GROUP_JID@g.us`, `YOUR_GROUP_JID@g.us`, `GROUP_ID@g.us`, `XXXXXXXXXX-XXXXXXXXXX@g.us`, `123456789@g.us`, `987654321@g.us` → `<group_jid>`
  - `tg:12345678` → `tg:<chat_id>`
  - `user@example.com` → `<example_email>`
  - `user@your-server.tail12345.ts.net` → `user@<tailnet_host>`
- **Files changed**: docs/concepts/multi-agent-architecture.md, docs/guides/whatsapp-bot.md, docs/reference/claude-code-settings.md, docs/getting-started/first-run-verification.md, docs/guides/remote-access.md, docs/reference/mcp-servers.md
- **Final scan**: No residual real identifiers or credentials detected; remaining `sk-ant-*` patterns are format-only with `...` truncation; `@g.us` in grep commands are shell pattern arguments (not JIDs)

## 2026-03-29 — First Elaboration Run (DoS: Docs on Steroids)

- **21 sessions processed** → **21 docs generated** across 5 categories
- **Getting Started (3)**: hardware-and-os-setup, authentication, first-run-verification
- **Guides (8)**: google-drive, remote-access, whatsapp-bot, telegram-bot, gws-cli, skills, 1password, cron-automation
- **Concepts (5)**: multi-agent-architecture, workspace-isolation, bootstrap-files, semantic-data-layer, agent-fleet-design
- **Reference (3)**: claude-code-settings, gws-cli-reference, mcp-servers
- **Troubleshooting (2)**: common-errors, integration-issues
- **index.yaml**: Updated with all 21 docs, summaries, tags, and paths
- **MkDocs Material**: Preview system configured with dark mode, search, navigation tabs
- **Homepage**: docs/index.md created with navigation cards
- **Language**: All docs in US English (translated from Italian source sessions)
- **Security**: PII scan performed — all sensitive data redacted in docs; sessions flagged for review

### Self-Assessment
- Session quality is excellent — rich in concrete examples, real commands, and error resolutions
- Italian-to-English translation was straightforward; technical terms stayed consistent
- Some sessions overlap (e.g., google-drive + google-account) — merged into single docs
- Entity graph and memory system concepts deserve deeper standalone docs in future runs
- Consider splitting the Telegram troubleshooting section once native threadId support arrives
- Sessions contain some real identifiers (Drive folder IDs, bot names) that need sanitization

---

## 2026-03-27

- **Project initialized**: Repository structure created
- **Templates**: Session template and validation schema defined
- **Docs structure**: Diátaxis-based documentation framework established
- **Sources**: Initial setup — no sessions processed yet

### Self-Assessment
- First run — structure is in place, awaiting first session contributions
- The session template may need refinement after processing the first few real sessions
- Consider adding example sessions to guide early contributors

## 2026-03-30 — Second Elaboration Run (Merge Pass)

- **21 session files processed** with **merge-first handling** against the existing 21-doc corpus
- **16 docs updated** with additional sources and version bumps where the session expanded an existing topic
- **5 sessions mapped as coverage-only merges** into already-established docs without creating duplicates
- **docs/index.yaml** refreshed to reflect the current frontmatter state and cumulative source coverage
- **Session statuses** flipped from `ready` to `processed` across the full March 12–28 batch
- **Language**: US English preserved throughout; no secrets or PII introduced

### Notes
- The corpus now reflects merged source provenance instead of one-doc-per-session duplication
- No new doc had a strong enough standalone signal to justify creating a duplicate topic file

## 2026-04-01 — Daily Elaboration Pass

- **7 sessions processed** → merged into existing guides/troubleshooting docs instead of creating duplicates
- **Guides updated**:
  - `cron-jobs-and-automation` now covers cost-aware cron model selection, subscription-backed providers, and explicit SKIP behavior for monitors/briefings
  - `1password-secrets-management` now documents subscription fallback keys as emergency-only secrets stored in 1Password
- **Troubleshooting updated**:
  - `common-errors` now includes fallback-API-key drift when subscription auth is unavailable
- **index.yaml**: refreshed generated timestamp, source coverage, and version counters for updated docs
- **Upstream check**: `openclaw --version` = `2026.3.31`, matching `docs/meta/upstream-version.yaml` (`last_known_version: 2026.3.31`) — no upstream discrepancy detected, no new release notes to summarize
- **Session statuses**: flipped all processed sessions from `ready` to `processed`

### Self-Assessment
- Good merge discipline: the new sessions mostly reinforced existing topics, so updating the canonical docs was cleaner than adding duplicate pages
- The doc corpus now reflects the current subscription-first workflow for cron jobs and secrets management
- Upstream consistency is clean; next run only needs a real release delta if OpenClaw moves past 2026.3.31
