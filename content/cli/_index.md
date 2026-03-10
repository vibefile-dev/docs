---
title: "CLI Reference"
description: "Complete reference for all vibe commands — init, run, list, check, and status."
weight: 4
---

## Overview

```sh
vibe init                         # detect project type and generate a Vibefile
vibe init --language <lang>       # use a specific language template
vibe init --force                 # overwrite an existing Vibefile
vibe run <target>                 # run a target and its dependencies
vibe run <target> --dry           # print what would be executed without running
vibe run <target> --recompile     # force LLM recompile for this target
vibe run <target> --recompile-all # recompile this target and all deps
vibe list                         # list all targets with their descriptions
vibe check                        # validate the Vibefile without running anything
vibe status                       # show compiled/uncompiled state of all targets
```

All commands are implemented and functional.

---

## `vibe init`

Bootstraps a new Vibefile by detecting project language, framework, and infrastructure from manifest files and generating targets for the detected stack. No LLM call required — templates are preconfigured.

**Flags:**
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
- Warns about targets using unimplemented modes (`@mcp`, `@skill`)

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

States include: compiled (valid cache), stale (inputs changed), not compiled, hand-edited (script modified outside of LLM), and agent/skill (not applicable for caching).

---

## Global flags

- `--api-key <key>` — API key for the LLM provider (overrides all other resolution)
- `--verbose` / `-v` — enable verbose debug logging via `slog`
