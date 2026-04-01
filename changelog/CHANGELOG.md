# Changelog

All elaboration runs by Kos are documented here.

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
