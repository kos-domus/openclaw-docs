---
title: "Gateway Media Policy and Trusted-Text Boundaries"
slug: media-attachment-policy
category: reference
tags: ["security", "media", "attachments", "gateway", "prompt-injection"]
sources:
- sessions/2026-11-06-decisore-locale-spesabot-pipeline-fase-v.md
last_updated: "2026-11-07"
version: 1
---

# Gateway Media Policy and Trusted-Text Boundaries

Reference notes on how media/attachment policy interacts with agent trust, and where untrusted text can enter an agent's context through messaging tools.

## Media policy: `strict: false` is broader than it looks

With a permissive media policy (`strict: false`), an agent can attach **nearly any file in the home directory** that is not on the denylist. Two consequences:

- the denylist is the only barrier, and it is a blocklist — everything unlisted is allowed
- **the allowlist overrides the denylist**: an entry present in both is allowed

Therefore grant allowlist entries by **exact path**, never by directory or pattern, and treat any broad allowlist entry as equivalent to disabling the denylist for that subtree.

## Inter-agent sends enter the destination context

Messaging tools that deliver a message into another agent's session (e.g. `hermes send`) mirror the sent message into the destination session **as the agent's own message**, with no opt-out. Any untrusted text embedded in such a send — quoted user input, scraped content, external API responses — is injected into the receiving agent's context with agent-level trust.

Rule: sanitize or summarize untrusted content before sending it inter-agent; never forward raw external text through a session-mirroring send.

## Checklist

- Prefer `strict: true` for agents that handle untrusted input.
- Allowlist by exact file path; re-audit entries whenever a directory moves.
- Assume everything an agent can attach is exfiltratable if the agent is prompted by untrusted content.
- Treat inter-agent messages as a prompt-injection surface, same as web content.
