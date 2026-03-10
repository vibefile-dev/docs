---
title: "Failure Handling"
description: "Exit code conventions, preflight checks, auto-retry on generation errors, and how failures interact with caching."
weight: 8
---

## Overview

Not all failures are equal. When a generated script exits with non-zero code, the CLI needs to know: was the script wrong, or did the script correctly detect a real problem? The answer determines whether to retry, cache, or report.

## Exit code convention

Generated scripts use exit codes as a protocol between the script and the CLI:

| Exit code | Meaning | CLI behaviour |
|-----------|---------|---------------|
| `0` | Success | Cache the script, report success |
| `1` | Task failed legitimately | Report failure, do NOT retry — script is correct but task found a real problem (tests fail, lint errors) |
| `2` | Precondition not met | Report missing requirement, do NOT retry — environment isn't ready |
| `≥ 3` | Script error / generation bug | Auto-retry with error context (up to `max_retries` attempts) |

The system prompt instructs the LLM to follow this convention when generating scripts. Exit 1 means "I ran the task correctly and it found a problem." Exit 2 means "I checked and a required tool or version is missing." Any other non-zero exit (3+) typically means the script itself is broken — a wrong command, bad flags, or a misunderstanding of the project.

## Preflight checks

The system prompt instructs the LLM to emit a **preflight section** at the top of every generated script. This section verifies that required tools and versions are available before doing anything, and exits with code 2 if something is missing:

```bash
#!/bin/bash
set -euo pipefail

# --- preflight ---
command -v go >/dev/null 2>&1 || { echo "error: go is required but not installed"; exit 2; }

# --- task ---
go test -race -v ./...
```

For tasks that require a specific version, the LLM may add version checks (e.g. from `go.mod`, `package.json`):

```bash
# --- preflight ---
command -v go >/dev/null 2>&1 || { echo "error: go is required but not installed"; exit 2; }
go_version=$(go version | grep -oP 'go\K[0-9]+\.[0-9]+')
[[ "$(printf '%s\n' "1.25" "$go_version" | sort -V | head -1)" == "1.25" ]] || { echo "error: go >= 1.25 required (found $go_version)"; exit 2; }
```

The LLM infers what to check from project context — `go.mod` tells it the Go version, `package.json` tells it the Node version, and so on. Preflight checks **verify but never install** — the script should not run `apt-get install` or `brew install` for system-level dependencies. If a tool is missing, the script reports what's needed and exits.

This gives the user a clear, actionable error message instead of a cryptic failure halfway through execution.

## Auto-retry on generation errors

When a script exits with 3+, the CLI assumes the script is wrong and retries:

```
vibe run build
  → generate script (attempt 1)
  → execute → exit code 127 (command not found)
  → retry: send error output back to LLM
    "The script failed with: line 5: pnpm: command not found
     The project uses npm, not pnpm. Regenerate the script."
  → generate script (attempt 2)
  → execute → exit code 0
  → cache the fixed script
```

The retry prompt includes:

- The original task recipe
- The script that failed
- The full stderr/stdout from the failed execution
- An instruction to fix the problem

Retry behaviour is configurable:

```makefile
max_retries = 2     # default: 1
```

```sh
vibe run build --no-retry   # disable retry, fail immediately
```

If all retries are exhausted, the CLI fails with the last error and does NOT cache the broken script.

## Caching and failure interaction

| Outcome | Cache the script? | Why |
|---------|:-----------------:|-----|
| Exit 0 — success | Yes | Script works |
| Exit 1 — legitimate failure | Yes | Script is correct; task found a problem (user fixes code and reruns) |
| Exit 2 — precondition not met | Yes | Script is correct; environment needs setup (user installs tool and reruns) |
| Exit 3+ — retry succeeds | Yes | Cache the **fixed** version |
| Exit 3+ — retries exhausted | No | Script is broken; don't persist |

Key insight: exit 1 and 2 scripts are **correct scripts**. Caching them means the next `vibe run` skips the LLM — the user fixes their code or installs the tool, and reruns instantly.

## Dependency failure

When a target's dependency fails, the default behaviour is to **stop the chain**. Remaining dependencies are skipped and the dependent target does not run.

Example:

```
vibe run deploy        # depends on: test, build
  → run test → exit 1 (tests fail)
  → skip build (dependency test failed)
  → skip deploy
  ✗ deploy failed: dependency "test" failed
```

This is the safe default. A `--continue-on-error` flag may be added in the future for CI scenarios where you want to run all independent targets and collect all failures.
