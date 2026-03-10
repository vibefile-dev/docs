---
title: "Project Detection"
description: "How vibe init detects your project and how to write custom language detectors and addon detectors."
weight: 5
---

## Overview

`vibe init` automatically detects your project's language, framework, and infrastructure — then generates a Vibefile with appropriate targets. No LLM call required.

Detection is built on two pluggable interfaces: **language detectors** identify the primary project type, and **addon detectors** find tools and platforms that contribute additional targets. Both use a registry pattern — detectors register themselves at import time and are run automatically.

This system is designed to be extended. You can contribute new detectors for additional languages, or write addon detectors for tools and platforms you use.

---

## How detection works

When you run `vibe init`, the CLI follows this flow:

```
vibe init
  1. Try language detection at the repo root
     → If a language matches → single-project mode
  2. If no match at root, scan immediate subdirectories
     → If sub-projects found → monorepo mode
  3. Run addon detectors (Docker, Helm, Makefile, etc.)
  4. Merge all targets into one Vibefile
  5. Write Vibefile
```

### Single-project mode

When a language detector matches at the repo root (e.g. finds `go.mod`), the CLI generates targets for that language and runs all addon detectors at the root:

```sh
$ vibe init
  ✓ Go project detected (go 1.25.0, module: github.com/myorg/myapp)
  ✓ Docker detected → docker
  ✓ Vibefile created with 9 targets
```

### Monorepo mode

When no language matches at the root, the CLI scans immediate subdirectories. Each detected sub-project gets its targets prefixed with the directory name:

```sh
$ vibe init
  ✓ Monorepo detected — 2 sub-projects found
  ✓ go/ — Go project (go 1.25.7)
  ✓ go/Docker detected → go-docker
  ✓ ui/ — Nextjs project (nextjs 15.1.0)
  ✓ Makefile detected → make-build, make-test, make-lint
  ✓ Vibefile created with 24 targets
```

Targets become `go-build`, `go-test`, `ui-dev`, `ui-lint`, etc. Recipes include directory context (e.g. *"in the go/ directory, compile the Go binary..."*). Addons are also scanned per-subdirectory — a `Dockerfile` inside `go/` produces `go-docker`.

Directories like `.git`, `node_modules`, `vendor`, `dist`, and hidden directories are skipped during scanning.

---

## Built-in language detectors

| Language | Detects | Targets generated |
|----------|---------|-------------------|
| **Go** | `go.mod`, `go.work` | `build`, `test`, `lint`, `fmt`, `vet`, `check`, `clean`, `install` |
| **Next.js** | `package.json` with `next` dependency | `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `e2e`, `check`, `clean`, `deps` |

The Go detector also handles **workspace projects** (`go.work`) — it parses all `use` directives, collects module paths, and generates recipes that reference all workspace modules.

The Next.js detector identifies the package manager (npm, pnpm, yarn, bun), TypeScript support, router type (App/Pages), test frameworks (vitest, jest, playwright, cypress), and Prettier.

---

## Built-in addon detectors

| Addon | Detects | Targets contributed |
|-------|---------|---------------------|
| **Docker** | `Dockerfile` | `docker` |
| **Fly.io** | `fly.toml` | `deploy` |
| **Vercel** | `vercel.json` | `deploy` |
| **Cloudflare** | `wrangler.toml` or `wrangler.json` | `cf-dev`, `cf-deploy` |
| **Helm** | `Chart.yaml` (root or subdirectories) | `helm-lint`, `helm-template`, `helm-package` |
| **Makefile** | `Makefile`, `GNUmakefile` | `make-<target>` for each discovered target |

The Makefile addon parses existing Makefile targets and wraps them — preferring `.PHONY` targets when available, and skipping internal targets (`all`, `default`, `_`-prefixed).

---

## Writing a language detector

A language detector identifies a project's primary language and tooling. It implements the `Detector` interface:

```go
package detect

type Detector interface {
    Name() string
    Detect(repoRoot string) (*ProjectInfo, bool)
}
```

`Name()` returns the language identifier (e.g. `"go"`, `"python"`). `Detect()` checks the given directory for language-specific files and returns a `ProjectInfo` with project metadata, or `false` if the language isn't detected.

### ProjectInfo

```go
type ProjectInfo struct {
    Language       string            // "go", "node", "python", "rust"
    Framework      string            // "gin", "next", "flask" (optional)
    PackageManager string            // "go", "npm", "pip", "cargo"
    Version        string            // language version from go.mod, .nvmrc, etc.
    BinaryName     string            // inferred from module path or package name
    Module         string            // go module path, npm package name, etc.
    Modules        []string          // workspace module directories (go.work)
    HasTests       bool              // test files detected
    Metadata       map[string]string // detector-specific extras
}
```

### Example: a minimal Python detector

```go
package python

import (
    "path/filepath"
    "github.com/vibefile-dev/vibe/detect"
)

func init() { detect.Register(&Detector{}) }

type Detector struct{}

func (d *Detector) Name() string { return "python" }

func (d *Detector) Detect(repoRoot string) (*detect.ProjectInfo, bool) {
    // Check for pyproject.toml, setup.py, or requirements.txt
    for _, file := range []string{"pyproject.toml", "setup.py", "requirements.txt"} {
        if detect.FileExists(filepath.Join(repoRoot, file)) {
            return &detect.ProjectInfo{
                Language:       "python",
                PackageManager: "pip",
                BinaryName:     filepath.Base(repoRoot),
                Metadata:       make(map[string]string),
            }, true
        }
    }
    return nil, false
}
```

The key points:

- **Package name** — put the detector in its own sub-package under `detect/` (e.g. `detect/python`)
- **`init()` function** — call `detect.Register(&Detector{})` to register at import time
- **`detect.FileExists()`** — helper for checking file presence
- **Return `false`** when no match — the registry moves on to the next detector

### Pairing with a template provider

A detector identifies the project. A **template provider** generates the actual Vibefile targets. Implement `TemplateProvider`:

```go
package detect

type TemplateProvider interface {
    Language() string
    Provide(project *ProjectInfo) *Template
}
```

Register it in `init()` with `detect.RegisterTemplate(&TemplateProvider{})`. The template provider receives the `ProjectInfo` populated by the detector and returns a `Template` with variables and targets.

### Example: Python template provider

```go
package python

import "github.com/vibefile-dev/vibe/detect"

func init() {
    detect.Register(&Detector{})
    detect.RegisterTemplate(&TemplateProvider{})
}

type TemplateProvider struct{}

func (p *TemplateProvider) Language() string { return "python" }

func (p *TemplateProvider) Provide(project *detect.ProjectInfo) *detect.Template {
    return &detect.Template{
        Variables: []detect.TemplateVariable{
            {Key: "model", Value: "claude-sonnet-4-6"},
            {Key: "name", Value: project.BinaryName},
        },
        Targets: []detect.TemplateTarget{
            {
                Name:    "test",
                Section: "build & test",
                Recipe:  "run all Python tests with pytest and verbose output",
            },
            {
                Name:    "lint",
                Section: "build & test",
                Recipe:  "run ruff or flake8 linting on the entire project",
            },
            {
                Name:    "fmt",
                Section: "build & test",
                Recipe:  "format all Python files using ruff format or black",
            },
            {
                Name:         "check",
                Section:      "quality gates",
                Dependencies: []string{"fmt", "lint", "test"},
                Recipe:       "print a summary of what passed — all quality gates complete",
            },
        },
    }
}
```

### Wiring it in

For the detector to run, it must be imported in `cmd/init.go` with a blank import:

```go
import (
    _ "github.com/vibefile-dev/vibe/detect/python"
)
```

This triggers the `init()` function which registers both the detector and template provider.

---

## Writing an addon detector

Addon detectors find tools, platforms, and infrastructure and contribute additional targets to the Vibefile. They implement the `Addon` interface:

```go
package detect

type Addon interface {
    Name() string
    Detect(repoRoot string) *AddonResult // nil if not detected
}
```

`Detect()` returns an `AddonResult` with a human-readable label and a list of targets, or `nil` if the tool isn't detected.

### AddonResult

```go
type AddonResult struct {
    Label   string           // human-readable label, e.g. "Docker", "Helm"
    Targets []TemplateTarget // targets contributed by this addon
}
```

### Example: a Terraform addon

```go
package terraform

import (
    "path/filepath"
    "github.com/vibefile-dev/vibe/detect"
)

func init() { detect.RegisterAddon(&Addon{}) }

type Addon struct{}

func (a *Addon) Name() string { return "terraform" }

func (a *Addon) Detect(repoRoot string) *detect.AddonResult {
    // Look for .tf files or terraform directory
    if !detect.FileExists(filepath.Join(repoRoot, "main.tf")) {
        return nil
    }

    return &detect.AddonResult{
        Label: "Terraform",
        Targets: []detect.TemplateTarget{
            {
                Name:    "tf-init",
                Section: "infrastructure",
                Recipe:  "run terraform init to initialize the working directory",
            },
            {
                Name:    "tf-plan",
                Section: "infrastructure",
                Recipe:  "run terraform plan and show the execution plan",
            },
            {
                Name:         "tf-apply",
                Section:      "infrastructure",
                Dependencies: []string{"tf-plan"},
                Recipe:       "run terraform apply to apply the planned changes",
            },
        },
    }
}
```

The same registration pattern applies — blank import in `cmd/init.go`:

```go
import (
    _ "github.com/vibefile-dev/vibe/detect/terraform"
)
```

### How addons merge

When both a language template and addons contribute targets, they're merged into a single Vibefile. If a target name conflicts (e.g. both the language template and an addon generate a `deploy` target), the first writer wins and the conflict is logged in debug output.

---

## YAML template overrides

You don't need to write Go code to customize the generated Vibefile. Templates can be overridden with YAML files:

- `.vibe/templates/<lang>.yaml` — project-local override
- `~/.vibe/templates/<lang>.yaml` — user-global override

These take priority over built-in templates.

### Template format

```yaml
variables:
  - key: model
    value: claude-sonnet-4-6
  - key: name
    value: my-project

targets:
  - name: build
    section: "build & test"
    recipe: "compile the project for production"

  - name: test
    section: "build & test"
    recipe: "run all tests with verbose output"

  - name: check
    section: "quality gates"
    dependencies: [test, build]
    recipe: "all quality gates passed"

  - name: deploy
    section: "infrastructure"
    recipe: "deploy to production"
    directives:
      - "@network outbound"
```

Each target can have:
- `name` — target name (required)
- `recipe` — plain-English recipe string (required)
- `section` — section header for grouping in the Vibefile
- `dependencies` — list of dependency target names
- `directives` — list of directive strings (e.g. `@network outbound`)

---

## Current detector layout

The detection code lives in the `detect/` package with each detector in its own sub-package:

```
detect/
├── detect.go           ← Detector and Addon interfaces, registry, scanning
├── project.go          ← ProjectInfo, SubProject, AddonResult types
├── template.go         ← Template types, resolution, generation
├── helpers.go          ← FileExists and shared utilities
├── golang/
│   ├── detect.go       ← Go language detector
│   └── template.go     ← Go template provider
├── nextjs/
│   ├── detect.go       ← Next.js language detector
│   └── template.go     ← Next.js template provider
├── docker/
│   └── addon.go        ← Docker addon
├── fly/
│   └── addon.go        ← Fly.io addon
├── vercel/
│   └── addon.go        ← Vercel addon
├── cloudflare/
│   └── addon.go        ← Cloudflare addon
├── helm/
│   └── addon.go        ← Helm addon
└── makefile/
    └── addon.go        ← Makefile addon
```

Each sub-package has an `init()` function that registers with the central registry. This keeps detectors self-contained and independently maintainable.
