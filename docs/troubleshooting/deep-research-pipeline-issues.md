---
title: "Deep Research Pipeline Issues"
slug: "deep-research-pipeline-issues"
category: "troubleshooting"
tags: ["deep-research", "troubleshooting", "mcp", "gemini", "zai"]
sources:
  - "sessions/2026-04-08-deep-research-pipeline-phase1-build.md"
  - "sessions/2026-04-08-deep-research-phases-2-3-security-review-2.md"
  - "sessions/2026-04-08-deep-research-glm51-zai-integration-4.md"
  - "sessions/2026-04-08-deep-research-p2-billing-gemini-testing-3.md"
  - "sessions/2026-04-09-deep-research-vision-ocr-glm-supermarket-test.md"
last_updated: "2026-04-14"
version: 1
---

# Deep Research Pipeline Issues

## `FIRECRAWL_API_KEY` works in your shell but fails in MCP

**Cause**: the MCP server process does not inherit the variable.

**Fix**: export it in the parent runtime and verify `.mcp.json` only references variables that already exist.

## Gemini CLI hangs or rejects flags

**Cause**: wrong invocation style.

**Fix**: use `-p` for prompts and prefer stdin piping for long prompts.

## GPT-5 style requests fail with `max_tokens not supported`

**Cause**: GPT-5.x expects `max_completion_tokens`.

**Fix**: change the request parameter.

## Z.AI says `Insufficient balance` even with a subscription

**Cause**: the request is hitting the regular endpoint instead of the coding endpoint.

**Fix**: use the coding endpoint for subscription-backed GLM access.

## A page has lots of text but extraction still finds no product data

**Cause**: the useful data is inside images or flyers.

**Fix**: use screenshot-based vision extraction and `forceVision: true` for known image-heavy sources.

## Jobs time out under realistic multi-site load

**Cause**: browser rendering plus LLM extraction plus retries exceed the timeout.

**Fix**: raise the job timeout and keep timeout handling explicit so failed jobs close resources cleanly.

## Gemini or Z.AI returns 429s intermittently

**Cause**: provider capacity, not necessarily your local setup.

**Fix**: keep the fallback chain intact and log the provider issue instead of treating it as a parsing bug.
