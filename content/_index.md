---
title: "Vibefile Documentation"
description: "The task runner for the AI era — declare what you want, let the machine figure out how."
---

## What is Vibefile?

A Vibefile is a task runner configuration file. It describes **what** tasks do and **when** they run — not **how** they are implemented. The *how* is figured out at runtime by an LLM that reads the Vibefile alongside relevant context from your repo.

The format is inspired by Makefiles. The dependency model is identical. The only thing that changes is the recipe: instead of shell commands, you write a plain-English description of intent.

```makefile
# Vibefile

model = claude-sonnet-4-6

build:
    "compile and bundle for production"

test:
    "run the full test suite with coverage"

deploy: test build
    "deploy to fly.io and verify the app came up healthy"
```

## How it works

1. You write a `Vibefile` with targets described in plain English
2. On `vibe run <target>`, the CLI collects relevant context from your repo
3. An LLM generates the implementation (a shell script)
4. The generated script is cached — subsequent runs are instant and free
5. Dependencies are resolved and executed in topological order

## The mental model

```
source code → Vibefile (intent) → compiled script (implementation)

first run:    Vibefile → LLM → .vibe/compiled/build.sh → execute
next runs:    Vibefile → .vibe/compiled/build.sh → execute (no LLM)
```

The Vibefile is your source code. The generated shell script is the binary. Just like a compiler, the expensive step happens once and the output is reused.

## Key concepts

| Concept | Description | Status |
|---------|-------------|--------|
| **Targets** | Named tasks with plain-English recipes and optional dependencies | Implemented |
| **Codegen mode** | LLM generates shell scripts from task descriptions | Implemented |
| **Compiled targets** | Generated scripts cached to disk — committed to git for free CI | Implemented |
| **Project detection** | Auto-detect project type and generate Vibefile with sensible targets | Implemented |
| **Agent mode** | LLM executes tasks via MCP server tool calls (`@mcp`) | Coming soon |
| **Skills** | Reusable, community-maintained task implementations (`@skill`) | Coming soon |
| **Sandbox** | All generated code runs in an isolated container | Coming soon |

## Quick start

```sh
# Install the CLI
go install github.com/vibefile-dev/vibe@latest

# Initialize a Vibefile for your project
vibe init

# Run a target
vibe run build

# See all targets
vibe list
```
