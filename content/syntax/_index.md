---
title: "Syntax"
description: "Variables, targets, dependencies, and recipes — the building blocks of a Vibefile."
weight: 3
---

## File name

The file must be named exactly `Vibefile` with no extension, placed at the repository root.

```
my-project/
└── Vibefile
```

## Variables

Variables use `KEY = value` syntax. Substitute them anywhere with `$(KEY)`:

```makefile
ENV = production

deploy:
    "deploy to $(ENV) on fly.io"
```

## Targets

A target has this structure:

```
target-name [: dep1 dep2 ...]:
    recipe
    [directives]
```

- **target-name** — lowercase, hyphens allowed, must be unique within the file
- **dependencies** — space-separated list of other target names that must complete first
- **recipe** — double-quoted plain English string
- **directives** — optional modifiers prefixed with `@`

## Dependencies

Dependencies are declared after the target name, separated by a colon:

```makefile
deploy: test build
    "deploy to production"
```

`deploy` will not run until `test` and `build` have both completed successfully. Identical to Makefile behaviour.

## Recipe

The recipe is a double-quoted plain English string. It is sent to the LLM along with repo context to generate the implementation at runtime:

```makefile
seed:
    "populate the database with realistic fake data for 10 users"
```

Multi-line recipes use continuation indentation:

```makefile
release: test build
    "bump the version, update the changelog,
     tag the commit, and push to origin"
```

## Model configuration

The model is declared at the top of the Vibefile. It is not a secret and belongs in the file:

```makefile
model = claude-sonnet-4-6

deploy:
    "deploy to production"
```

Individual targets can override the top-level model:

```makefile
model = claude-haiku-4-5

release: test build
    model = claude-opus-4-6
    "bump version, update changelog, tag, push, open release PR"
```

## Complete example

```makefile
# Vibefile

model   = claude-sonnet-4-6
project = my-saas-app
env     = production

# ── tasks ────────────────────────────────────────────────

build:
    "compile and bundle the project for $(env)"

test:
    @skill python-test

seed: build
    "populate the database with realistic fake data for 10 users"

migrate: build
    "run any pending database migrations safely"

deploy: test build
    "deploy to $(env) on fly.io and verify the app came up healthy"
    @require clean git status
    @mcp fly-mcp

review:
    "open a PR, write a summary of changes, request review from the team"
    @mcp github-mcp

release: test build
    @skill python-publish
    "also update the changelog and tag the commit"
    @require clean git status
```
