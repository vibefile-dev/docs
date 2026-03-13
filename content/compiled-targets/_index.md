---
title: "Compiled Targets"
description: "How generated scripts are cached, invalidated, and committed for free CI runs."
weight: 7
---

## The compile-once model

Running the LLM on every `vibe run` would be expensive and slow. Vibefile uses **target compilation**: the first time a codegen target runs, the generated shell is saved to disk. Subsequent runs execute the saved shell directly — no LLM call, no latency, no API cost.

**Mental model:** the first `vibe run build` is the compile step. Every run after that executes the compiled output. The Vibefile is source code. The generated shell script is the binary.

```
first run                          subsequent runs
──────────────────────────────     ───────────────────────────
vibe run build                     vibe run build
  → collect repo context             → load .vibe/compiled/build.sh
  → call LLM API          (cost)     → execute script             (free)
  → save .vibe/compiled/build.sh
  → execute script
```

---

## Cache location

Compiled scripts live in `.vibe/compiled/`. This directory **must be committed to version control.**

**Why commit:**

- **CI runs are free and fast.** CI uses the cached script — no LLM, no API key, no latency.
- **One person pays the cost.** Dev runs `vibe run` (or `--recompile`), the LLM generates the script, it's committed with the Vibefile change. All subsequent runs are free.
- **Auditability.** Code review shows both intent change (recipe) and implementation change (compiled shell).
- **Reproducibility.** The same script runs everywhere. No variance from different LLM responses.

```
my-project/
├── Vibefile
└── .vibe/
    ├── config.yaml
    ├── compiled/
    │   ├── build.sh
    │   ├── build.lock
    │   ├── test.sh
    │   └── test.lock
    └── skills/
```

Do **not** add `.vibe/compiled/` to `.gitignore`. The `vibe init` command generates an appropriate `.gitignore` that keeps compiled output tracked.

---

## Cache invalidation

A target's compiled output is invalidated when its inputs change. The `.lock` file stores a checksum of everything sent to the LLM:

```yaml
# .vibe/compiled/build.lock
recipe: "compile and bundle the project for production"
model: claude-sonnet-4-6
context_files:
  package.json: sha256:a1b2c3...
  tsconfig.json: sha256:d4e5f6...
  Vibefile: sha256:7g8h9i...
variables:
  env: production
skill_hash: sha256:j0k1l2...    # present only when @skill is used
script_hash: sha256:m3n4o5...
generated_at: 2025-03-08T14:22:00Z
```

On each run, the CLI recomputes the checksum and compares it to the `.lock` file. If anything changed — recipe, variable, config file, model — the target recompiles. Otherwise the cached script runs directly. This mirrors how Turborepo and Bazel handle input hashing, applied to LLM-generated code.

### What triggers a recompile

| Change | Recompiles? |
|--------|:-----------:|
| Recipe string edited | Yes |
| Variable value changed | Yes |
| Relevant context file changed (e.g. `package.json`) | Yes |
| Model version changed | Yes |
| Skill `SKILL.md` content changed | Yes |
| Unrelated source files changed | No |
| Running on a different machine (same inputs) | No |

---

## Codegen and skill targets only

Compilation applies to **codegen** and **skill** mode targets. Both generate shell scripts that are cached for subsequent runs. Agent targets (`@mcp`) are inherently dynamic — they interact with live services, inspect state, and adapt at runtime. Caching their output would be meaningless and potentially dangerous.

```makefile
build:
    "compile and bundle for production"
    # compiled — same commands every time

test:
    @skill go-test
    "run the Go tests for this project"
    # compiled — skill content is hashed, recompiles when SKILL.md changes

deploy: build
    "deploy to production on fly.io and verify health"
    @mcp fly-mcp
    # not compiled — agent adapts to live state each run
```

---

## Force recompile

```sh
vibe run build --recompile       # force LLM call for this target
vibe run build --recompile-all   # force LLM call for this target and all deps
```

---

## Reviewing compiled output

Because compiled scripts are committed, they're auditable. Code review shows both intent (recipe) and implementation (shell). Teams catch problems before production.

Hand-editing compiled scripts is supported: the CLI detects manual modification (checksum mismatch without recipe change) and warns but still executes. Use `--recompile` to regenerate from the recipe.

---

## Ejecting to a Makefile

If you want to leave Vibefile behind entirely, or need a plain Makefile for environments without the vibe CLI, use `vibe eject`:

```sh
vibe eject -o Makefile
```

This generates a standard GNU Make-compatible Makefile by inlining all compiled scripts as recipes. The result requires no vibe CLI, no LLM, and no API key. See the [CLI Reference](/cli/#vibe-eject) for full details.
