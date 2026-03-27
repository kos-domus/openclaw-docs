# OpenClaw Docs — Kos Elaboration Engine

This project is a **living documentation system** for OpenClaw (Claude Code).
Kos reads raw work sessions and produces structured, analytical documentation.

## Your Role

You are the **documentation elaboration engine**. Your job:

1. **Read** new or updated session files in `sessions/`
2. **Analyze** the content: extract features, configs, patterns, gotchas, solutions
3. **Generate or update** structured docs in `docs/`
4. **Update** the machine-readable index at `docs/index.yaml`
5. **Log** what you did in `changelog/CHANGELOG.md`

## Elaboration Rules

### Reading Sessions
- Sessions are markdown files in `sessions/` with YAML frontmatter
- Check the `status` field: only process sessions with `status: ready`
- After processing, update the session's status to `status: processed`
- Each session has tags — use them to route content to the right doc section

### Writing Docs
- Follow the **Diátaxis** framework:
  - `docs/getting-started/` — First-time setup, installation, basic usage
  - `docs/guides/` — Task-oriented how-to guides (step-by-step)
  - `docs/reference/` — Exhaustive reference (settings, skills, hooks, MCP, API)
  - `docs/concepts/` — Architecture, mental models, design decisions
  - `docs/troubleshooting/` — Known issues, error messages, solutions
- Every doc file MUST have YAML frontmatter:
  ```yaml
  ---
  title: "Human-readable title"
  slug: "unique-kebab-case-id"
  category: "getting-started|guides|reference|concepts|troubleshooting"
  tags: ["hooks", "mcp", "skills"]
  sources: ["sessions/2026-03-27-hooks-setup.md"]
  last_updated: "2026-03-27"
  version: 1
  ---
  ```
- Use clear H2/H3 hierarchy for scannability
- Include practical code examples whenever possible
- Cross-reference related docs using relative links
- Keep each doc focused on ONE topic — split if it grows beyond ~500 lines

### Updating Docs
- When new session data overlaps with existing docs, **merge** — don't duplicate
- Increment the `version` field when updating
- Add the new session to the `sources` array
- Preserve information from previous versions unless explicitly contradicted

### Index Management
- After every elaboration run, regenerate `docs/index.yaml`
- The index must list every doc with: slug, title, category, tags, path, last_updated
- This index is the **primary entry point for programmatic fetching**

### Changelog
- Append to `changelog/CHANGELOG.md` with format:
  ```
  ## YYYY-MM-DD
  - **Added**: new-doc-slug — Brief description
  - **Updated**: existing-doc-slug — What changed
  - **Sources**: list of session files processed
  ```

### Quality Standards
- Be analytical, not just descriptive — explain WHY things work, not just HOW
- Flag contradictions between sessions (config X worked in session A but not B)
- When uncertain, mark with `> ⚠️ **Unverified**: ...` blockquote
- Prefer concrete examples over abstract explanations

## Self-Improvement Protocol

After each elaboration run, append a brief self-assessment to `changelog/CHANGELOG.md`:
- What went well in this elaboration
- What was ambiguous or hard to categorize
- Suggestions for improving session format or doc structure

This feedback will be reviewed and used to refine these instructions over time.

## File Naming Conventions
- Sessions: `YYYY-MM-DD-short-description.md`
- Docs: `slug-matching-frontmatter.md` (e.g., `hooks-configuration.md`)
- Keep names lowercase, kebab-case, no spaces

## Do NOT
- Delete or overwrite session files (they are the source of truth)
- Invent information not present in sessions
- Change the directory structure without explicit approval
- Process sessions with `status: draft` — those are work in progress
