---
title: "Directives"
description: "Modifiers that change how a target is executed — @require, @mcp, @skill, @network, and @no-sandbox."
weight: 10
---

## Overview

Directives are optional modifiers on a target. They appear indented below the recipe, one per line, prefixed with `@`.

```makefile
deploy: test build
    "deploy to production on fly.io"
    @require clean git status
    @mcp fly-mcp
```

Directives determine execution mode, preconditions, network access, and sandbox behaviour. Each directive has a specific purpose and can be combined as needed.

All directives are **parsed and validated** by the CLI today. Enforcement depends on the directive — see the status notes on each section below.

---

## `@require`

> **Coming soon.**

---

## `@mcp`

> **Coming soon.** Codegen mode (the default, without `@mcp`) is fully functional.

---

## `@skill`

> **Coming soon.**

---

## `@network`

> **Coming soon.**

---

## `@no-sandbox`

> **Coming soon.**

---

## Execution modes

Directives determine how a target runs:

| Mode | Trigger | Behaviour | Status |
|------|---------|-----------|--------|
| **Codegen** | Plain recipe, no `@mcp` | LLM generates a shell script; CLI executes it | **Implemented** |
| **Agent** | `@mcp` present | LLM executes via tool calls to declared MCP servers | Coming soon |
| **Skill** | `@skill` present | Instructions come from `SKILL.md`; mode is then codegen or agent depending on other directives | Coming soon |
