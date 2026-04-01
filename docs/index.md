# OpenClaw Docs

**Community-driven, AI-elaborated documentation for [Claude Code](https://claude.ai/code) (OpenClaw).**

This documentation is generated from real work sessions — hands-on experiences setting up, configuring, and operating OpenClaw in production environments. Every guide, concept, and troubleshooting entry is grounded in actual usage, not hypothetical examples.

---

## Quick Navigation

<div class="grid cards" markdown>

-   **Getting Started**

    ---

    Hardware setup, authentication, and first-run verification.

    [:octicons-arrow-right-24: Start here](getting-started/hardware-and-os-setup.md)

-   **Guides**

    ---

    Step-by-step how-tos for Google Drive, WhatsApp, Telegram, 1Password, and more.

    [:octicons-arrow-right-24: Browse guides](guides/google-drive-integration.md)

-   **Concepts**

    ---

    Multi-agent architecture, workspace isolation, bootstrap files, and fleet design.

    [:octicons-arrow-right-24: Understand the architecture](concepts/multi-agent-architecture.md)

-   **Reference**

    ---

    Settings, GWS CLI commands, MCP servers, and configuration options.

    [:octicons-arrow-right-24: Look it up](reference/claude-code-settings.md)

-   **Troubleshooting**

    ---

    Common errors and integration issues with solutions from real deployments.

    [:octicons-arrow-right-24: Fix an issue](troubleshooting/common-errors.md)

</div>

---

## How This Documentation Works

```
Contributors write session logs  →  sessions/
                ↓
    Kos reads & analyzes (cron)
                ↓
    Structured docs generated    →  docs/
                ↓
    Index updated                →  docs/index.yaml
```

**21 session logs** processed into **21 structured documents** across 5 categories.

## Programmatic Access

Fetch the machine-readable index:

```bash
curl -s https://raw.githubusercontent.com/kos-domus/openclaw-docs/main/docs/index.yaml
```

Use in your `CLAUDE.md`:

```markdown
# OpenClaw Reference
Fetch https://raw.githubusercontent.com/kos-domus/openclaw-docs/main/docs/index.yaml
for the documentation index. Fetch individual docs by path as needed.
```

## Contributing

We welcome session contributions from anyone using Claude Code. See the [Contributing Guide](https://github.com/kos-domus/openclaw-docs/blob/main/CONTRIBUTING.md) for details.
