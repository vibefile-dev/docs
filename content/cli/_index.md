---
title: "CLI Reference"
description: "Complete reference for all vibe commands — init, run, list, check, status, and eject."
weight: 4
---

## Overview

```sh
vibe init                         # detect project type and generate a Vibefile
vibe init --empty                 # create a minimal skeleton Vibefile
vibe init --language <lang>       # use a specific language template
vibe init --force                 # overwrite an existing Vibefile
vibe run <target>                 # run a target and its dependencies
vibe run <target> --dry           # print what would be executed without running
vibe run <target> --recompile     # force LLM recompile for this target
vibe run <target> --recompile-all # recompile this target and all deps
vibe list                         # list all targets with their descriptions
vibe check                        # validate the Vibefile without running anything
vibe status                       # show compiled/uncompiled state of all targets
vibe eject                        # generate a standalone Makefile from compiled scripts
vibe eject -o Makefile            # write ejected Makefile to a file
vibe eject --compile              # compile missing targets first, then eject
```

All commands are implemented and functional.

---

## `vibe init`

Bootstraps a new Vibefile by detecting project language, framework, and infrastructure from manifest files and generating targets for the detected stack. No LLM call required — templates are preconfigured.

**Flags:**
- `--empty` — create a minimal skeleton Vibefile without auto-detected targets (for manual setup)
- `--language <lang>` — skip auto-detection, use specific language template (e.g. `go`, `nextjs`)
- `--force` — overwrite existing Vibefile

### How detection works

`vibe init` uses a pluggable registry of **language detectors** and **addon detectors** that run independently. Results are merged into a single Vibefile.

**Built-in language detectors:**

| Language | Detects | Key features |
|----------|---------|-------------|
| **Go** | `go.mod`, `go.work` | Module path, Go version, workspace support, test detection |
| **Next.js** | `package.json` with `next` dep | Package manager, TypeScript, router type, test frameworks |

**Built-in addon detectors:**

| Addon | Detects | Targets contributed |
|-------|---------|---------------------|
| **Docker** | `Dockerfile` | `docker` |
| **Fly.io** | `fly.toml` | `deploy` |
| **Vercel** | `vercel.json` | `deploy` |
| **Cloudflare** | `wrangler.toml` or `wrangler.json` | `cf-dev`, `cf-deploy` |
| **Helm** | `Chart.yaml` | `helm-lint`, `helm-template`, `helm-package` |
| **Makefile** | `Makefile`, `GNUmakefile` | `make-<target>` for each discovered target |

A Go project with a Dockerfile gets Go targets (build, test, lint, etc.) plus a `docker` target automatically.

### Monorepo support

When no language is detected at the root, `vibe init` scans immediate subdirectories. Each sub-project's targets are prefixed with the directory name:

```sh
$ vibe init
  ✓ Monorepo detected — 2 sub-projects found
  ✓ go/ — Go project (go 1.25.7)
  ✓ go/Docker detected → go-docker
  ✓ ui/ — Nextjs project (nextjs 15.1.0)
  ✓ Makefile detected → make-build, make-test, make-lint
  ✓ Vibefile created with 24 targets
```

This generates targets like `go-build`, `go-test`, `ui-dev`, `ui-lint`, `make-build`, etc. Recipes include directory context (e.g. *"in the go/ directory, compile the Go binary..."*).

Addons are also scanned per subdirectory — a `Dockerfile` inside `go/` produces `go-docker`.

### Template overrides

Templates can be customized without modifying the CLI:
- `.vibe/templates/<lang>.yaml` — project-local override
- `~/.vibe/templates/<lang>.yaml` — user-global override
- Built-in templates — always available as fallback

See [Project Detection](/project-detection/) for details on writing custom detectors and templates.

---

## `vibe run`

The main command. Runs a target and all its dependencies in topological order.

**Flags:**
- `--dry` — print generated script without executing
- `--recompile` — force LLM regeneration for this target (ignores cache)
- `--recompile-all` — force LLM regeneration for this target AND all dependencies
- `--no-retry` — disable auto-retry on script generation errors
- `--cached-only` — only use cached scripts, never call LLM (auto-activated when `CI=true`)

**Execution flow:**

1. Parse Vibefile, resolve dependencies (topological sort, cycle detection)
2. For each target in dependency order:
   - Evaluate `@require` preconditions (fail fast if unmet)
   - Resolve `@skill` if present (load `SKILL.md`, show description)
   - Check compiled cache — if valid, use cached script
   - If no cache (or invalidated), collect context and call LLM
   - Execute script (currently on host; sandbox coming soon)
   - Cache result if appropriate per [exit code convention](/failure-handling/#exit-code-convention)
3. On script generation errors (exit 3+), auto-retry with error context fed back to the LLM

**CI mode:** When the `CI=true` environment variable is set (or `CI=1`), `--cached-only` is automatically enabled. If any target has a stale or missing cache, the run fails immediately with a clear error suggesting to recompile locally and commit.

> **Note:** Scripts currently run directly on the host via `bash`, not in a container sandbox. A warning is displayed at runtime. The `--dry` flag is useful for reviewing generated scripts before execution.

---

## `vibe list`

Lists all targets defined in the Vibefile with their recipe descriptions and dependencies.

```sh
$ vibe list
  build     "compile and bundle for production"
  test      "run all tests with coverage"
  deploy    test build → "deploy to fly.io and verify health"
```

---

## `vibe check`

Validates the Vibefile without running anything. Checks:

- Syntax is valid (parsing, quoting, structure)
- All dependency references resolve to defined targets
- No dependency cycles exist
- Model is specified
- Warns about targets using unimplemented modes (`@mcp`)

```sh
$ vibe check
  ✓ Vibefile is valid (5 targets, 0 warnings)
```

---

## `vibe status`

Shows the compiled/uncompiled state of all codegen targets.

```sh
$ vibe status
  build   ✓ compiled (2 hours ago)
  test    ✓ compiled (2 hours ago)
  deploy  · agent (not compiled)
  lint    ✗ stale (recipe changed)
```

States include: compiled (valid cache), stale (inputs changed), not compiled, hand-edited (script modified outside of LLM), and agent (not applicable for caching). Skill targets are compiled and tracked like codegen targets.

---

## `vibe eject`

Generates a standard, self-contained Makefile from the Vibefile and its compiled shell scripts. The resulting Makefile is 100% compatible with GNU Make and requires no vibe CLI, no LLM, and no API key to run.

This is the escape hatch: if a team decides to stop using Vibefile, or needs a plain Makefile for environments where the vibe CLI isn't available, `vibe eject` produces one that reproduces the exact same behaviour as the compiled targets.

**Flags:**
- `-o <file>` / `--output <file>` — write the Makefile to a file instead of stdout
- `--compile` — compile any missing or stale codegen targets before ejecting (requires an API key)

```sh
vibe eject                       # print generated Makefile to stdout
vibe eject -o Makefile           # write to a file
vibe eject --compile -o Makefile # compile everything first, then eject
```

When `--compile` is passed, each codegen target that has no compiled script (or a stale one) is compiled via the LLM before generating the Makefile. Targets that already have a valid cached script are skipped. This is a one-shot way to go from a fresh Vibefile to a complete Makefile without running each target individually.

**What gets ejected:**

- All codegen targets with compiled scripts in `.vibe/compiled/` are inlined as Makefile recipes
- Dependencies are preserved as Makefile prerequisites
- Vibefile variables are mapped to uppercased Makefile variables (e.g. `env` becomes `ENV`)
- Shebang lines and `set -euo pipefail` are stripped — the generated Makefile sets `SHELL := /bin/bash` and `.SHELLFLAGS := -euo pipefail -c` globally
- A `help` target is generated listing all available targets
- All targets are marked `.PHONY`

**What gets skipped:**

- **Agent targets** (`@mcp`) — these require live MCP server interaction and cannot be reduced to a shell script. A comment is left in the Makefile explaining the skip.
- **Targets without compiled scripts** — a warning is printed to stderr suggesting you run `vibe eject --compile` or `vibe run <target>` first.

**Example:**

```sh
$ vibe eject -o Makefile
Makefile written to Makefile
```

The generated Makefile:

```makefile
# Makefile — ejected from Vibefile by `vibe eject`
# This is a standalone Makefile. It does not require the vibe CLI.
SHELL := /bin/bash
.SHELLFLAGS := -euo pipefail -c
.ONESHELL:

ENV := production
PROJECT := my-saas-app

.DEFAULT_GOAL := build
.PHONY: build test deploy

# compile and bundle the project for production
build:
	echo "Building..."
	go build -o bin/app ./cmd/app

# run all tests with verbose output
test:
	go test -v -race ./...

# deploy: skipped (agent mode — requires @mcp, cannot be ejected)

help:
	@echo "Available targets:"
	@echo "  build                compile and bundle the project for production"
	@echo "  test                 run all tests with verbose output"
```

---

## Global flags

- `--api-key <key>` — API key for the LLM provider (overrides all other resolution)
- `--verbose` / `-v` — enable verbose debug logging via `slog`
