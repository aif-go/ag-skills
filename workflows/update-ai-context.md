# Sync AI Context to Existing Project

## Trigger

User asks to "update/sync ai-context", "refresh project instructions", or skill ai-context was just modified and needs deployment.

## Pre-check

Confirm current directory is an ag-core project (`go.mod` + `idl/api/`). If not, stop and inform user.

## Workflow

### 1. Copy ai-context files

```bash
SKILL_DIR="<this-skill-root>"     # directory containing SKILL.md
mkdir -p .claude/ai-context
cp "$SKILL_DIR/ai-context/"*.md .claude/ai-context/
```

### 2. Update CLAUDE.md

- If `.claude/CLAUDE.md` exists: preserve sections under `## Project-*` or `## Service Inventory`, replace all other content with the standard template (see [create-project.md](create-project.md) Step 4)
- If not exists: create from the same template

### 3. Report

```
✅ ai-context synced to .claude/ai-context/ (00-instructions, workflows, patterns, tools)
✅ CLAUDE.md updated (project sections preserved)
```
