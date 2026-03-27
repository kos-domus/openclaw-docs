You are generating a session log for the openclaw-docs knowledge base. Review the ENTIRE current conversation and produce a structured session file.

## Instructions

1. **Analyze the conversation** — identify the main topics, decisions, commands run, errors encountered, and key discoveries. Focus on anything related to OpenClaw, Claude Code, AI agents, automation, infrastructure, or tooling.

2. **Filter out noise** — skip greetings, small talk, off-topic tangents, and anything not useful for documentation.

3. **Generate the session file** with this exact format:

```markdown
---
title: "[Descriptive title of what was accomplished]"
date: "YYYY-MM-DD"
author: "kos-domus"
status: "ready"
tags: ["tag1", "tag2"]
openclaw_version: ""
environment:
  os: ""
  ide: ""
  model: ""
---

## Objective
[What was the goal of this session?]

## Context
[Starting state, constraints, relevant background]

## Steps Taken
### 1. [Action]
[What was done, commands, configs]
**Result**: [What happened]

[...more steps...]

## Configuration Changes
[List of config changes made]

## Key Discoveries
- [Non-obvious findings, gotchas, patterns]

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|----------|
| ... | ... | ... |

## Final State
[End result of the session]

## Open Questions
- [Unresolved items]
```

4. **CRITICAL SECURITY**: Before writing the file, scan your own output for:
   - Real phone numbers, email addresses, IP addresses → replace with placeholders
   - API keys, tokens, credentials → redact with `...`
   - Telegram IDs, WhatsApp JIDs, Drive folder IDs → use generic placeholders
   - If the conversation contains PII from the user, sanitize it

5. **Choose tags** from: hooks, skills, mcp, mcp-servers, settings, permissions, memory, context, claude-md, git, github, ci-cd, testing, debugging, automation, cron, scheduling, vscode, jetbrains, cli, web, desktop, remote, tailscale, setup, configuration, troubleshooting, performance, security, api, agent-sdk, multi-agent

6. **Write the file** to `~/openclaw-docs/sessions/YYYY-MM-DD-short-description.md` using today's date

7. **Git add, commit, and push**:
```bash
cd ~/openclaw-docs && git add sessions/ && git commit -m "Add session: [short description]" && git push
```

8. **Confirm** to the user with the filename and a one-line summary.

If the conversation has no OpenClaw-relevant content, tell the user "Nothing to save — no OpenClaw-relevant content in this session."
