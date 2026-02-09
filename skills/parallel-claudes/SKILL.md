---
name: parallel-claudes
description: >
  Guide for running multiple parallel Claude Code sessions using cw (Claude Worktree Manager).
  Use when the user wants to parallelize work across multiple Claude instances, manage git worktrees
  for concurrent AI coding tasks, or merge results back together. Triggers on tasks involving
  parallel Claude sessions, worktree management, or splitting work across multiple agents.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Parallel Claudes with `cw`

`cw` (Claude Worktree Manager) is a CLI tool that uses git worktrees to spin up isolated working directories so you can run multiple parallel Claude Code sessions against the same repository without interference.

## When to Apply

- The user wants to run multiple Claude Code sessions in parallel on the same repo
- A task can be broken down into independent subtasks that don't conflict
- The user wants to create isolated branches for different features/fixes simultaneously
- The user needs to merge parallel work back together via PRs or local squash merges

## Prerequisites

- **Git** must be installed
- **Claude Code CLI** (`claude`) must be available
- **GitHub CLI** (`gh`) must be installed (required for PR-based merge)
- **`cw` must be installed** — if not, install with:

```bash
curl -fsSL https://raw.githubusercontent.com/joyco-studio/cw/main/install.sh | bash
```

After installing, restart your shell so the `cw` function and tab completions are loaded.

## Commands Reference

| Command | Description |
|---|---|
| `cw new <name> [flags] [prompt]` | Create a new worktree and optionally launch Claude Code with a prompt |
| `cw open <name> [prompt]` | Open Claude Code in an existing worktree |
| `cw ls` | List all active `cw` worktrees with branch and commits-ahead count |
| `cw cd <name>` | Change directory into a worktree |
| `cw merge <name>` | Push branch and create a GitHub PR |
| `cw merge <name> --local` | Squash-merge the branch locally (no PR) |
| `cw rm <name>` | Remove a worktree and its branch |
| `cw clean` | Remove all `cw` worktrees |
| `cw upgrade` | Self-update to the latest release |

### Flags for `cw new`

- `--open` / `-o` — Open Claude Code immediately after creating the worktree
- `--no-install` — Skip automatic dependency installation
- `--no-open` — Create the worktree without opening Claude Code

## Workflow

### 1. Break Down the Task

Identify independent subtasks that can be worked on in parallel without conflicting file changes. Each subtask gets its own worktree and Claude Code session.

### 2. Create Worktrees

```bash
# Create worktrees with descriptive names and prompts
cw new auth --open "implement OAuth2 login flow"
cw new api --open "build REST API endpoints for user management"
cw new tests "add unit tests for the auth module"
```

Each worktree:
- Creates an isolated copy of the repo at `.worktrees/<name>`
- Creates a branch named `cw/<name>`
- Auto-installs dependencies (detects npm, yarn, pnpm, pip)
- Optionally launches Claude Code with the given prompt

### 3. Monitor Progress

```bash
# See all active worktrees and how many commits each has
cw ls
```

### 4. Open or Resume Sessions

```bash
# Open Claude Code in an existing worktree
cw open tests "now add integration tests too"

# Navigate into a worktree manually
cw cd auth
```

### 5. Merge Results

```bash
# Create a GitHub PR from the worktree branch
cw merge auth

# Or squash-merge locally without a PR
cw merge api --local
```

### 6. Clean Up

```bash
# Remove a specific worktree
cw rm tests

# Remove all worktrees
cw clean
```

## Key Behaviors

- **Branch namespacing**: All branches are prefixed with `cw/` to avoid collisions
- **Base branch detection**: Automatically finds `main` or `master`
- **Gitignore**: Automatically adds `.worktrees/` to `.gitignore`
- **Safety**: Warns before removing worktrees with unmerged changes
- **Dependency install**: Detects lockfiles and runs the right package manager

## Example: Parallelizing a Feature

```bash
# Split a large feature into three parallel tasks
cw new feat-ui --open "build the settings page UI components"
cw new feat-api --open "create the settings API routes and handlers"
cw new feat-db --open "add database migrations and models for settings"

# Check progress
cw ls

# Merge each piece as it completes
cw merge feat-db
cw merge feat-api
cw merge feat-ui

# Clean up
cw clean
```

## Troubleshooting

- **`cw cd` doesn't work**: Make sure you've sourced `cw` in your shell config (`source ~/.local/bin/cw` in `.zshrc`/`.bashrc`) and restarted your shell
- **Merge conflicts**: Resolve manually in the worktree before running `cw merge`
- **`gh` not found**: Install the GitHub CLI — `brew install gh` on macOS
- **Uncommitted changes**: `cw merge` will refuse to merge if there are uncommitted changes in the worktree; commit or stash first

## Reference

- Repository: https://github.com/joyco-studio/cw
