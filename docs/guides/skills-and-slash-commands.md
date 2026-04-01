---
title: Skills and Slash Commands
slug: skills-and-slash-commands
category: guides
tags:
- skills
- clawhub
- configuration
- security
sources:
- sessions/2026-03-22-top-skills-and-updates.md
last_updated: '2026-03-30'
version: 2
---

# Skills and Slash Commands

Skills extend OpenClaw's capabilities by providing agents with instructions, tool access, and workflow patterns. This guide covers the skill ecosystem, top skills, and how to evaluate them safely.

## What Is a Skill?

A skill is essentially a **set of instructions** (SKILL.md) that tells an agent how to use specific tools. Skills do not grant capabilities by themselves — they require:

1. **Config**: The relevant tools must be enabled (e.g., `exec` for running commands)
2. **Bridge tool**: The underlying tool must be installed on the machine
3. **Authorization**: External service credentials must be configured

## Installing Skills

### From ClawHub

```bash
clawhub install skill-name
```

### Manual Installation (Symlink)

```bash
ln -s /path/to/skill-directory ~/.openclaw/skills/skill-name
```

### Per-Agent vs Global Skills

| Location | Scope |
|----------|-------|
| `~/.openclaw/skills/` | Global — available to all agents |
| `~/.openclaw/workspace-<agent>/skills/` | Agent-specific — only that agent can use it |

## Top Skills by Download Count

| Skill | Downloads | Purpose |
|-------|-----------|---------|
| **GOG** (Google Workspace) | 14,000+ | Gmail, Calendar, Drive, Docs, Sheets, Contacts |
| **Summarize** | 10,000+ | Summarize URLs, YouTube videos, podcasts, local files |
| **Composio** | High | 860+ external tools (GitHub, Slack, Gmail) with transparent OAuth handling |
| **Exa** | High | Developer-focused search (GitHub repos, docs, forums — not SEO blogs) |
| **n8n Workflow** | High | Bridge between OpenClaw conversations and n8n automation workflows |
| **Obsidian** | High | RAG over Obsidian vaults — natural language queries on your notes |
| **ElevenLabs** | High | TTS, speech-to-text, voice cloning, phone call failsafe |

### Bundled Skills

Some skills come bundled with OpenClaw:
- **Gemini**: Coding assistance + Google search
- **Peekaboo**: macOS screenshot capture

## Evaluating Skill Safety

> ⚠️ **Approximately 80% of skills on ClawHub are low-quality or potentially malicious.** In early 2026, a campaign called "ClawHavoc" planted malicious skills with names nearly identical to legitimate ones.

### Before Installing Any Skill

1. **Read the code** — always inspect the skill's source before enabling it
2. **Check the publisher** — prefer the 53 official bundled skills or well-known publishers
3. **Review permissions** — understand what tools the skill requires
4. **Test in sandbox** — run the skill in sandbox mode first

### SKILL.md Keyword Specificity

Skills with vague descriptions in their SKILL.md file may not be invoked correctly by the model. If a skill isn't being triggered:

- **Bad**: "Get information about something"
- **Good**: "Retrieve a detailed company dossier from Crunchbase"

Specific keywords help the model understand when to invoke the skill.

## Notable Skill Deep-Dives

### Composio

Provides access to 860+ external tools through a single integration. Handles OAuth transparently — the user talks in chat, Composio connects to the services.

**Best for**: Multi-service integrations where managing individual OAuth flows would be impractical.

### Exa

A search index built for developers. Returns official documentation, real GitHub repositories, and technical forums — not SEO-optimized blog posts.

**Best for**: Reducing model hallucinations by providing real technical context.

### n8n Workflow

Bridges conversational AI (OpenClaw) with structured automation (n8n).

**Gotcha**: If OpenClaw chats normally instead of invoking n8n, the SKILL.md description is probably too vague. Add specific keywords.

### Obsidian

Transforms an Obsidian vault from a static archive into a reasoning system (RAG on personal notes).

**Limitation**: If OpenClaw runs in a sandbox or container, it needs filesystem access to the vault. Cloud-based alternatives (like Notion) may work better in containerized setups.

### ElevenLabs

Three variants available:
- `elevenlabs-tts` — Basic text-to-speech
- `elevenlabs-voices` — 18 personas, 32 voices
- `elevenlabs-cli` — Full suite: TTS + STT + voice cloning

Supports inline directives to change voice mid-response.

## OpenClaw Updates (v2026.3.13)

Key features relevant to skills:

- **ContextEngine plugin slot**: Full lifecycle hooks (bootstrap, ingest, assemble, compact, afterTurn) for custom memory strategies
- **Bundle discovery/install**: Compatible with Codex, Claude, and Cursor
- **Chrome DevTools MCP attach mode**: Attach to Chrome sessions with real login credentials
- **SSRF fail-closed policy**: Edge cases are blocked by default
- **Hardened webhook security**

## What's Next

- [Claude Code Settings Reference](../reference/claude-code-settings.md) — Configuration options
- [Common Errors](../troubleshooting/common-errors.md) — Troubleshooting skill issues
