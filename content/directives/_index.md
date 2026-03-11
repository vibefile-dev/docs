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

Asserts a precondition before the target runs. If the condition isn't met, the target fails immediately with a clear error — no LLM call, no script execution.

```makefile
deploy: test build
    "deploy to production on fly.io"
    @require clean git status
    @require on branch main
```

**Recognised patterns:**

| Condition pattern | What it checks |
|-------------------|---------------|
| `clean git status`, `clean working directory`, `no uncommitted changes` | `git status --porcelain` is empty |
| `on branch main`, `branch release` | Current git branch matches the expected name |
| `passing tests`, `tests pass` | Deferred — passes with a note (enforced by dependency chain) |
| `<tool> installed`, `<tool> is installed` | `command -v <tool>` succeeds |
| Anything else | Passes with a "skipped" note (unrecognised conditions don't block) |

Multiple `@require` directives can appear on the same target. All are evaluated; all must pass.

```makefile
release: test build
    "tag and push a release"
    @require clean git status
    @require on branch main
```

---

## `@mcp`

> **Coming soon.** Codegen mode (the default, without `@mcp`) is fully functional.

---

## `@skill`

Delegates a target's implementation to a **skill** — a reusable `SKILL.md` file containing domain-specific instructions for the LLM.

```makefile
test:
    @skill go-test
    "run the Go tests for this project"
```

When a target has `@skill`, the CLI:

1. **Resolves** the skill by searching for `SKILL.md` in this order:
   - `skills/<name>/SKILL.md` (project-local)
   - `.vibe/skills/<name>/SKILL.md` (project .vibe directory)
   - `~/.vibe/skills/<name>/SKILL.md` (user-global)
   - Additional paths from `skill_sources` in `.vibe/config.yaml`
2. **Provides** the skill to the LLM as a tool via the Anthropic tool-calling API
3. The LLM **invokes** the skill tool during generation to load the full instructions
4. Generates a shell script following both the recipe and the skill's instructions

### `SKILL.md` format

A skill file uses YAML frontmatter for metadata and markdown for instructions:

```markdown
---
description: Run Go tests with race detection, coverage reporting, and clear output formatting
---

# Go Test Skill

Run all Go tests for the project with best practices.

## Requirements

- Go toolchain must be installed
- Project must have a valid `go.mod`

## Instructions

1. Run tests across all packages with race detection enabled (`-race`)
2. Enable verbose output (`-v`)
3. Generate a coverage profile to `coverage.out`
4. After tests complete, print a one-line coverage summary

## Example command

\`\`\`bash
go test -race -v -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1
\`\`\`
```

The `description` in the frontmatter is shown in CLI output when the skill is loaded and is included in the tool description sent to the LLM.

### Cache invalidation

Skill content is included in the cache hash. When a `SKILL.md` file changes, any target using that skill is automatically recompiled.

### Limitations

- Skills currently require an **Anthropic model** (Claude). Using `@skill` with an OpenAI model produces an error.
- A target can reference one `@skill`. The recipe is optional but recommended — it gives the LLM additional task-specific context on top of the skill instructions.

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
| **Skill** | `@skill` present | LLM instructions come from `SKILL.md`; uses Anthropic tool-calling to deliver skill content | **Implemented** |
