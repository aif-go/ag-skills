# Create New ag-core Project

## Trigger

User asks to create a new ag-core project, scaffold a microservice, or init an ag-core workspace.

## Workflow

### Step 1: Gather Input (MUST confirm)

Ask the user for all three inputs. Show suggested defaults but require explicit confirmation. **Do NOT proceed to Step 2 until the user confirms.**

| Input | Question | Suggested |
|-------|----------|-----------|
| Project name | "项目名是什么？" | — |
| Template repo | "模板仓库 URL？" | `http://gitlab.allinfinance.com/aifgo/ag-layout-demo.git` |
| Template branch | "模板分支？" | `base` |

Confirm: "使用模板 `<repo>` @ `<branch>` 创建项目 `<name>`，确认吗？"

### Step 2: Run aggo new

```bash
aggo new -r <template-url> -b <branch> <project-name>
```

If `aggo` is not found, guide user: `go install <aggo-module>@latest`

### Step 3: Inject AI Context

Create `.claude/` directory in the new project and copy ai-context files from the skill:

```bash
cd <project-name>
mkdir -p .claude/ai-context

# Copy from skill root's ai-context/ directory
SKILL_ROOT="<path-to-ag-skills>"
cp "$SKILL_ROOT/ai-context/00-instructions.md" .claude/ai-context/
cp "$SKILL_ROOT/ai-context/workflows.md"      .claude/ai-context/
cp "$SKILL_ROOT/ai-context/patterns.md"       .claude/ai-context/
cp "$SKILL_ROOT/ai-context/tools.md"          .claude/ai-context/
```

> **Note**: The SKILL_ROOT is the directory where `SKILL.md` resides. Use the actual path.

### Step 4: Create .claude/CLAUDE.md Entry Point

Write `.claude/CLAUDE.md` with the following content:

```markdown
# Project Instructions

This is an ag-core microservice project. Follow the instructions in `.claude/ai-context/`.

## File Priority
1. `.claude/ai-context/00-instructions.md` — Core rules and decision tree
2. `.claude/ai-context/workflows.md` — Standard workflows
3. `.claude/ai-context/tools.md` — aggo CLI reference
4. `.claude/ai-context/patterns.md` — Code patterns quick reference

## Quick Rules
- **Proto-First**: define `.proto` before writing any code
- **Never edit** generated code in `adpgen/` or `svcgen/`
- **Business logic** in `internal/biz/` (delegate from `internal/service/`)
- **Post-generation**: always `go mod tidy && go build ./...`
- **-m flag**: only affects `kitex`/`hertz` plugins (server|client), has no effect on `go`/`api`/`server`/`service`

## aggo Quick Commands

    # Full server generation (recommended one-liner)
    aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto

    # Generate client code (calling other services)
    aggo proto -p kitex,hertz -m client -e ./idl/api ./idl/api/<svc>/<svc>.proto

    # Post-generation
    go mod tidy && go build ./...
```

### Step 5: Verify

```bash
cd <project-name>
go mod tidy
go build ./...
```

### Step 6: Report Result

Inform the user:

- Project created at `<cwd>/<project-name>`
- Template: `<template-url>` @ `<branch>`
- AI context injected to `.claude/`
- Directory structure
- Next steps:
  1. Define API: create `.proto` in `idl/api/<service>/`
  2. Generate code: `aggo proto -p go,api,server,kitex,hertz,service -m server ...`
  3. Implement logic: `internal/biz/` for business logic, `internal/service/` for thin layer
  4. Build & run: `cd cmd/server && go build && ./server`

## Resulting .claude/ Structure

```
.claude/
├── CLAUDE.md                      # Entry point (navigation + quick rules)
└── ai-context/
    ├── 00-instructions.md          # Decision tree + rules
    ├── workflows.md                # 6 standard workflows
    ├── patterns.md                 # Code patterns
    └── tools.md                    # aggo CLI reference
```
