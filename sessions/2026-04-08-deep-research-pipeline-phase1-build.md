---
title: "Deep Research Pipeline Phase 1 — MCP server, Firecrawl fetcher, multi-site scraping"
date: "2026-04-08"
author: "kos-domus"
status: "ready"
tags: ["mcp", "mcp-servers", "automation", "configuration", "multi-agent", "setup"]
openclaw_version: "2026.4.5"
environment:
  os: "Ubuntu 24.04 (AceMagic mini PC)"
  ide: "VS Code Remote"
  model: "claude-opus-4-6"
---

## Objective
Build Phase 1 of a domain-agnostic deep research pipeline as an internal tool for all OpenClaw agents. Strategic evaluation via Co-CEO agent first, then architecture design, then implementation.

## Context
- Kos wanted to evaluate whether an ETL web research pipeline should be the first project
- Originally framed as potential SaaS product, reframed as internal tool during Co-CEO evaluation
- All main agents (Kai, Mr Wolf, Kos, Master Control) need access to deep research capability
- Master Control is the primary user for work/consulting purposes
- Pipeline must be domain-agnostic: same quality regardless of topic (crypto, backend tech, family decisions)
- n8n was considered and rejected as orchestration layer
- Codex (OpenAI) to be used for site-specific extractors (leaf implementations)

## Steps Taken

### 1. Co-CEO strategic evaluation
Asked the Co-CEO agent to evaluate the ETL pipeline as first project. Initial recommendation: NO-GO as SaaS, limited GO for internal use.

After clarifying it's strictly an internal tool (not a product), Co-CEO revised to **Full GO** with phased build plan:
- Phase 1 (weeks 1-3): Working pipeline + 2 domains, Firecrawl only
- Phase 2 (weeks 4-6): Protocol store, scheduling, Kai notifications
- Phase 3 (post-baby): Agent-initiated research, LLM-assisted extraction

**Result**: Clear strategic direction with hard deadline (Phase 1 before Chris arrives June 2026).

### 2. n8n evaluation (Co-CEO + Orchestration Architect)
Both agents independently evaluated n8n and agreed: **don't use it**.
- n8n adds 300-500MB RAM overhead on a mini PC with 16GB
- Creates "orchestrator-of-orchestrators" anti-pattern with OpenClaw
- Scraping logic ends up in Code nodes with worse debugging than native IDE
- Recommended instead: CLI tool + file-based queue + systemd daemon (~50MB)

**Result**: n8n rejected. Custom lightweight architecture chosen.

### 3. Orchestration Architect designed multi-agent access
Designed tiered access control for the research pipeline:

| Tier | Agent | Behavior |
|------|-------|----------|
| Full | Master Control, Kos (CoS) | 3 concurrent jobs, no approval |
| Standard | Mr Wolf | 1 concurrent job, no approval |
| Request | Kai | Needs CoS approval before execution |

Recommended MCP server as integration point (native agent access), not just CLI.

**Result**: Architecture approved with CLI + MCP server dual interface.

### 4. Indexed official docs and openclaw-docs into permanent memory
Fetched and indexed:
- MCP TypeScript SDK docs (McpServer, StdioServerTransport, registerTool, Zod schemas)
- MCP architecture docs (protocol, tools, transports, lifecycle)
- OpenClaw architecture reference (agent fleet, tool access patterns, config files)
- OpenClaw docs full index (all 21 docs mapped by category)

Created 3 new reference memory files, updated ETL project memory with revised scope.

**Result**: Permanent knowledge base for all future sessions.

### 5. Built Phase 1 deep-research pipeline
Full implementation in TypeScript:

```
tools/deep-research/
├── src/
│   ├── types.ts              # Job, config, result type definitions
│   ├── config.ts             # Domain config loader, access control, paths
│   ├── pipeline.ts           # Job executor: fetch → normalize → store
│   ├── mcp-server.ts         # MCP server (7 tools for agent access)
│   ├── cli.ts                # CLI wrapper for direct invocation
│   ├── fetchers/firecrawl.ts # Firecrawl API integration
│   ├── normalizer/normalize.ts # Keyword-based structured extraction
│   └── queue/job-queue.ts    # File-based job queue with tier enforcement
├── configs/
│   ├── crypto.yaml           # 5 default sites
│   ├── backend-tech.yaml     # 5 default sites
│   └── access.yaml           # Agent tier configuration
├── research/                 # Output: /research/{project}/{domain}/{date}/
└── queue/                    # Job state: jobs.json
```

MCP tools registered: `research_submit`, `research_status`, `research_results`, `research_list`, `research_domains`, `research_cancel`, `research_approve`

**Result**: Clean TypeScript build, zero errors.

### 6. End-to-end testing
Single-site test: DefiLlama → 1/1 OK in 5s, captured $91.5B TVL + full protocol rankings.

Multi-site test (9 sites including DL News, Decrypt, The Block, Daily Degen Substack):
- First attempt via MCP tool: 0/9 failed — Firecrawl 401 Unauthorized
- Root cause: `FIRECRAWL_API_KEY` env var not available to MCP server process
- Fix: Updated `.bashrc` to source `op-env-cached.sh` instead of `op-env.sh`
- Second attempt via CLI with sourced env: **9/9 OK in 5 seconds**

| Site | Fetch Time | Markdown Size |
|------|-----------|---------------|
| DefiLlama | 4.3s | 10,000 chars |
| CoinGecko | 1.1s | 10,000 chars |
| Dune | 0.8s | 6,056 chars |
| Messari | 1.2s | 10,000 chars |
| CryptoPanic | 1.2s | 10,000 chars |
| DL News | 1.1s | 10,000 chars |
| Decrypt | 0.9s | 10,000 chars |
| The Block | 0.9s | 10,000 chars |
| Daily Degen | 3.0s | 7,899 chars |

**Result**: Pipeline fully operational. Parallel Firecrawl fetching works.

## Configuration Changes
- `~/job-desk/tools/deep-research/` — new: entire deep-research pipeline project
- `~/job-desk/.mcp.json` — added `deep-research` MCP server entry
- `~/.bashrc` — switched from `op-env.sh` to `op-env-cached.sh` (with fallback)
- `~/.local/bin/research` — symlink to CLI for global access
- Memory files updated: `project_etl_web_research.md` (revised scope), `reference_mcp_sdk_typescript.md` (new), `reference_openclaw_architecture.md` (new), `reference_openclaw_docs_index.md` (new)

## Key Discoveries
- **MCP server env vars require parent process to have them exported**: The `${VAR}` syntax in `.mcp.json` resolves from the spawning process's environment, not from any config file. If the shell profile doesn't export the var, MCP tools fail with auth errors.
- **`.bashrc` non-interactive guard blocks env loading**: The `case $- in *i*) ;; *) return;; esac` at the top of `.bashrc` means non-interactive shells (like MCP server spawns) skip everything. The env vars must be available through another mechanism (e.g., systemd environment, or the MCP server's own env config).
- **Firecrawl parallel fetching is fast**: 9 sites in 5 seconds total, all parallel. No rate limiting issues on Hobby plan for this volume.
- **`import.meta.dirname` works on Node 24.14+**: No need for `fileURLToPath` workaround.
- **n8n is wrong for single-operator local systems**: Both Co-CEO and Orchestration Architect independently concluded this — it's designed for connecting SaaS APIs visually, not for compute-heavy local scraping pipelines.
- **DefiLlama has its own MCP server**: Discovered during scraping — could be used as an alternative data source in Phase 2.
- **Phase 1 normalizer is basic**: Keyword-based extraction captures some fields but misses most structured data from JS-heavy sites. LLM-assisted extraction (Phase 3) will dramatically improve this.

## Errors & Solutions
| Error | Cause | Solution |
|-------|-------|---------|
| Firecrawl 401 Unauthorized (all 9 sites) | `FIRECRAWL_API_KEY` not in MCP server process environment | Updated `.bashrc` to source `op-env-cached.sh`; MCP server env config in `.mcp.json` references `${FIRECRAWL_API_KEY}` |
| CLI showing `resultPath: undefined` | Reading job object before `updateJob()` persists the result path | Added `getJob(job.id)` call after `executeJob()` to get updated state |

## Final State
- **Deep research pipeline**: Phase 1 complete, TypeScript, 7 MCP tools, CLI, 2 domain configs
- **MCP server**: Registered in `.mcp.json`, available to Claude Code agents on session restart
- **CLI**: Globally accessible via `research` command
- **Test results**: 9/9 crypto sites scraped successfully in 5s, structured results stored
- **Agent access tiers**: Configured (full/standard/request) with approval workflow for Kai
- **Env fix**: `.bashrc` now sources cached secrets for reliable MCP server startup

## Open Questions
- MCP server env var loading in non-interactive shells — may need additional fix beyond `.bashrc` (e.g., systemd environment for OpenClaw gateway MCP)
- Phase 2 scope: protocol store format, scheduling mechanism (cron vs daemon polling), Kai notification integration path
- Firecrawl credit consumption tracking — need to monitor Hobby plan limits with multi-site jobs
- DefiLlama MCP server — evaluate as alternative/complement to scraping their site
- LLM-assisted extraction (Phase 3) — which model for cost-effective structured extraction from markdown?
