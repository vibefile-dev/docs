---
title: "Getting Started"
description: "Install the Vibefile CLI and create your first Vibefile in minutes."
weight: 2
---

## Installation

Vibefile is a Go-based CLI. Install it with:

```sh
go install github.com/vibefiledev/vibe@latest
```

You need **Go 1.21 or later**. If you've got that, you're in business.

## API key setup

The CLI needs an API key to call the LLM. You never put it in your Vibefile — resolution happens at runtime in this order:

1. `--api-key` CLI flag
2. `VIBE_API_KEY` environment variable
3. Provider-specific env var (inferred from the model you use):
   - `ANTHROPIC_API_KEY` for Claude
   - `OPENAI_API_KEY` for OpenAI
4. `~/.vibeconfig` global config file

Most people use `~/.vibeconfig` and forget about it:

```yaml
# ~/.vibeconfig — never committed to version control
default_model: claude-sonnet-4-6
anthropic_key: sk-ant-...
openai_key: sk-...
```

One file, both providers, done.

## Initialize a project

From your project root:

```sh
vibe init
```

This auto-detects your project type from manifest files (`go.mod`, `package.json`, `Cargo.toml`, etc.) and generates a Vibefile with sensible defaults. No LLM call — it's template-based.

Override detection with `--language` if you want a specific stack, or `--force` to clobber an existing Vibefile.

## Your first Vibefile

Here's the simplest thing that works:

```makefile
model = claude-sonnet-4-6

build:
    "compile and bundle for production"

test:
    "run all tests with verbose output"

check: test build
    "all quality gates passed"
```

Targets. Descriptions. Dependencies. That's it — the LLM figures out the how.

## Running targets

```sh
vibe run build          # run build
vibe run check          # runs test, then build, then check
vibe run build --dry    # preview the generated script without executing
```

The mental model:

```
first run:   vibe run build → LLM call → script generated and cached
next runs:   vibe run build → instant (served from cache, no LLM)
```

The expensive step happens once. After that you're just running shell scripts.

## What's next

- **[Syntax](/syntax/)** — variables, targets, dependencies, recipes
- **[CLI Reference](/cli/)** — all commands: init, run, list, check, status
- **[Project Detection](/project-detection/)** — how `vibe init` detects your project and how to extend it
