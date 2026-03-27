# Contributing to OpenClaw Docs

Thank you for helping build the community knowledge base for Claude Code!

## How to Contribute

### 1. Write a Session Log

A session log documents your real-world experience working with Claude Code. It captures what you did, what worked, what didn't, and what you learned.

**Steps:**
1. Fork this repository
2. Copy `sessions/_template.md` to a new file: `sessions/YYYY-MM-DD-short-description.md`
3. Fill in the template with your session details
4. Set `status: ready` in the frontmatter when complete
5. Open a Pull Request

### 2. Session Quality Guidelines

**Good sessions include:**
- A clear objective (what were you trying to do?)
- Concrete steps with actual commands/configs
- Results for each step (what happened?)
- Key discoveries (what wasn't obvious?)
- Errors and their solutions

**Sessions should NOT include:**
- API keys, tokens, or credentials
- Personal or sensitive data
- Proprietary business logic
- Speculation without evidence (use "## Open Questions" for hypotheses)

### 3. Tagging

Use tags from `sessions/schema.yaml` to categorize your session. Tags help Kos route your knowledge to the right documentation section.

Common tag combinations:
- Setting up hooks: `["hooks", "configuration", "setup"]`
- MCP server issue: `["mcp", "mcp-servers", "troubleshooting"]`
- VS Code integration: `["vscode", "setup", "configuration"]`
- Custom skill creation: `["skills", "configuration"]`

### 4. Review Process

1. **Automated validation**: PR checks verify frontmatter schema and required sections
2. **Human review**: A maintainer reviews for quality and safety (no secrets, no harmful content)
3. **Merge**: Once approved, your session is merged with `status: ready`
4. **Elaboration**: Kos processes your session in the next daily run
5. **Docs updated**: Your knowledge is integrated into the structured docs

### 5. What Happens to Your Session

Kos will:
- Extract factual knowledge from your session
- Merge it with existing documentation (no duplicates)
- Credit your session in the doc's `sources` frontmatter
- Never delete or modify your original session file (except setting `status: processed`)

## Reporting Issues

If you find errors in the generated docs:
- Open an issue describing the error
- Reference the doc slug and the specific section
- If possible, link to the session that may have caused the error

## Code of Conduct

- Be respectful and constructive
- Share real experiences, not speculation
- Don't include harmful, illegal, or deceptive content
- Respect Anthropic's usage policies

## Questions?

Open an issue or start a discussion. We're building this together.
