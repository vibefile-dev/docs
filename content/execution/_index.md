---
title: "Execution"
description: "Execution modes, the sandbox model, context collection, and how generated code runs safely."
weight: 6
---

## Execution modes

Each target runs in one of three modes, determined by its directives:

| Mode | Trigger | Behaviour | Status |
|------|---------|-----------|--------|
| **Codegen** | plain recipe, no `@mcp` | LLM generates a shell script; CLI executes it | **Implemented** |
| **Agent** | `@mcp` present | LLM executes via tool calls to declared MCP servers | Coming soon |
| **Skill** | `@skill` present | LLM instructions come from `SKILL.md`; mode is then codegen or agent depending on other directives | Coming soon |

**Codegen mode** is the core execution path today. The CLI collects repo context, calls the LLM to generate a shell script, and executes it. The generated script is cached for subsequent runs.

---

## Execution sandbox

> **Coming soon.** Scripts currently execute directly on the host machine via `bash`. A warning is displayed at runtime. Use `--dry` to review generated scripts before execution.

---

## Context collection

The CLI collects repo context before making the LLM call. The collector is task-aware — it sends only what's relevant for the declared intent. This is fully implemented.

**Always included:**

- Top-level file tree (depth 2)
- Contents of `Vibefile`

**Conditionally included** (detected from task name/intent):

- `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` — framework and dependency detection
- `fly.toml`, `Dockerfile`, `railway.json` — deploy-type tasks
- Test config files (`jest.config.*`, `pytest.ini`, etc.) — test-type tasks
- Schema files — seed/migration tasks
- Existing `Makefile` or shell scripts — inherit known-good patterns
- `git status` output — uncommitted changes, current branch

**Never included:**

- `node_modules/`, `.venv/`, build output directories
- Full source file contents unless directly relevant
- Secret or credential files

The goal: the LLM generates correct commands on the first attempt because it understands the repo, without being sent the entire codebase.

The context collector also produces file hashes used for [cache invalidation](/compiled-targets/#cache-invalidation) — when a relevant context file changes, the cached script is automatically recompiled.
