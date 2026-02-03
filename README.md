# Pipelex Skills

Skills for building [Pipelex](https://github.com/Pipelex/pipelex) workflows with Claude Code.

## Prerequisites

- Python 3.10+
- [Pipelex CLI](https://github.com/Pipelex/pipelex) installed

**Option A: Global install**
```bash
pip install pipelex
```

**Option B: Project dependency with uv**
```bash
uv add pipelex
```

Verify: `pipelex --version` or `uv run pipelex --version`

## Installation

This is a plugin repository containing Pipelex skills. To use it:

**Option A: Clone and reference locally**
```bash
git clone https://github.com/Pipelex/skills.git
claude --plugin-dir /path/to/skills
```

**Option B: Manual skill access**
The skills are documented in `skills/*/SKILL.md` and can be referenced directly.

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| Build | `/pipelex:build` | Create new Pipelex workflow bundles (.plx files) |
| Edit | `/pipelex:edit` | Modify existing workflow bundles |
| Check | `/pipelex:check` | Validate and report issues (read-only) |
| Fix | `/pipelex:fix` | Automatically fix validation errors |

## Usage Examples

### Build a new workflow

```
/pipelex:build
```

Creates a new .plx file with guided requirements gathering or direct creation.

### Edit an existing workflow

```
/pipelex:edit
```

Modify pipes, concepts, or structure in an existing .plx file.

### Check for issues

```
/pipelex:check
```

Validate a workflow and report errors, warnings, and suggestions without making changes.

### Fix issues automatically

```
/pipelex:fix
```

Automatically fix common validation errors and re-validate until the workflow passes.

## About Pipelex

Pipelex is an open-source language for building composable AI workflows. It uses `.plx` files to define typed concepts and pipes that transform data through focused, sequential steps.

Learn more at [docs.pipelex.com](https://docs.pipelex.com/pre-release)
