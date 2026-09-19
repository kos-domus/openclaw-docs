# Upstream Tracking — Extended-Stable Retraction 2026.7.33 → 2026.6.35

- **Channel**: npm `extended-stable` dist-tag + GitHub releases
- **Event**: dist-tag REGRESSION 2026.7.33 → **2026.6.35** (rollback to June LTS line)
- **Observed**: 2026-09-19 ~07:36 CEST by the KB daily run
- **Previous state (2026-09-18)**: extended-stable advanced 2026.6.35 → 2026.7.33 (npm publish 05:10 UTC, GitHub release v2026.7.33 05:33 UTC). Tracked in `2026-09-18-v2026.7.33-extended-stable.md`.
- **Current state (2026-09-19)**: npm `view openclaw dist-tags` → `extended-stable: 2026.6.35`; GitHub `releases/tags/v2026.7.33` → **404 Not Found**; `v2026.7.33` absent from `releases?per_page=10/30` lists. The release page and the dist-tag alias have both been withdrawn within ~24h of publication.

## Classification

This is a **full retraction**, not propagation lag: the 2026-09-18 run verified the GitHub release live (05:33 UTC publish, body read in full, 15.9 KB) and the paginated list no longer shows it. The npm package `openclaw@2026.7.33` still resolves in registry (`dist.tarball` present) — installed copies keep working, but the alias no longer points there and the canonical release page is gone.

## Operator-relevant highlights

- Anyone who pinned `extended-stable` or installed 2026.7.33 yesterday is now on an orphaned version: no release page, no alias pointing at it. If the July LTS line is re-published later, treat it as a fresh advance (re-verify from scratch), not a restoration.
- The 2026-09-18 artifact's highlights (security/credential hardening, channel delivery integrity, UTF-16 boundaries — 126 PRs) describe a version upstream has now withdrawn; read them as "July LTS line content", not as shipping guidance.
- Root cause is unknown from public signals (no advisory, no blog post, no deprecation note found). Watch whether 2026.7.x reappears; do not install it opportunistically.

## Relevance to this fleet

- Local OpenClaw CLI is 2026.6.8 (June line) — coincidentally now back in line with the restored extended-stable alias. No action required; the fleet does not pin LTS.
- Hermes: unaffected.

## Verification (2026-09-19)

- npm `view openclaw dist-tags` → `extended-stable: 2026.6.35` (live 07:36 CEST).
- GitHub `releases/tags/v2026.7.33` → HTTP 404 with body `{"message":"Not Found"}`.
- `releases?per_page=10` → top stable entries: v2026.9.5, v2026.9.4, v2026.6.35, v2026.9.3... — no v2026.7.33.
- npm `view openclaw@2026.7.33` → still resolves (`version = '2026.7.33'`, tarball URL present) — registry artifact survives, alias does not.
