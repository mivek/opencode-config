---
name: worktree-workflow
description: Per-feature git worktree workflow. Load before starting any non-trivial feature, refactor, or experiment. Each feature gets its own isolated worktree, allowing parallel development without branch-switching. Defines when to create, name, sync, and clean up worktrees.
compatibility: opencode
metadata:
  workflow: worktree-driven-development
---

# Worktree-driven workflow

This project uses **one worktree per feature**. Every non-trivial change happens in an isolated worktree, never directly on `main` or the current branch.

## Why worktrees

- **Parallel work without context-switching.** Switching branches with uncommitted changes is risky; worktrees let you keep multiple WIPs alive simultaneously.
- **Agent isolation.** Multiple AI sessions can work on different features without stepping on each other's files.
- **No "git stash dance".** You never need to stash to try something — just spin up a worktree.
- **Easy rollback.** A worktree that didn't pan out is deleted in one command, no commits to revert.

## When to create a worktree

**Create a new worktree** when the work matches any of these criteria :
- The change touches multiple files
- The change might fail or get abandoned
- You're experimenting with an approach you're not 100% sure of
- The work will take more than ~10 minutes
- You want to run tests/build without polluting your current state

**Do NOT create a worktree** for :
- One-line fixes you'll commit immediately
- Read-only investigation (use the current worktree)
- Looking up information (no code change)

## Naming conventions

Branches in worktrees follow this pattern :

```
<type>/<short-description>
```

Where `<type>` is one of :
- `feature/` — new functionality (e.g., `feature/user-export-csv`)
- `fix/` — bug fix (e.g., `fix/null-deref-in-auth`)
- `refactor/` — internal restructuring (e.g., `refactor/extract-payment-service`)
- `chore/` — maintenance, deps, build (e.g., `chore/upgrade-quarkus-3.16`)
- `spike/` — exploratory, may be thrown away (e.g., `spike/try-tigerbeetle`)
- `docs/` — documentation only (e.g., `docs/api-reference`)

The short description : lowercase, kebab-case, 2-5 words max. Be specific. `feature/user-stuff` is bad, `feature/user-csv-export` is good.

## Standard workflow

### 1. Create the worktree

Call the `worktree_create` tool with the branch name :

```
worktree_create:
  branch: "feature/user-csv-export"
  baseBranch: "main"   # optional, defaults to current HEAD
```

This will :
1. Create a git worktree under `~/.local/share/opencode/worktree/<project-id>/feature/user-csv-export`
2. Sync files per `.opencode/worktree.jsonc` (env files, symlink node_modules, etc.)
3. Run postCreate hooks (e.g., `./mvnw compile`, `pnpm install`, `terraform init`)
4. **Spawn a new terminal** with OpenCode running in the worktree

You continue in the original session OR switch to the new terminal — your choice. The new terminal has a fresh OpenCode session in the worktree directory.

### 2. Work in the worktree

In the worktree's terminal, everything you do is isolated. You can :
- Edit files freely
- Run tests
- Commit and amend
- Force-push (it's your branch)
- Break things (the other worktrees are untouched)

### 3. Decide the outcome

When the work is done, choose one of three paths :

#### A. Merge it (success)
```bash
# In the worktree
git push origin feature/user-csv-export

# In your main worktree (different terminal)
git checkout main
git pull
git merge feature/user-csv-export      # or via PR/MR on your forge
```

Then delete the worktree (see step 4).

#### B. Abandon it (didn't work out)
Just delete the worktree (step 4). The branch remains in git; you can clean it up later with `git branch -D feature/user-csv-export` if you're sure.

#### C. Pause it (will resume later)
Just leave it. Worktrees persist across reboots. You can have 10+ active simultaneously.

### 4. Delete the worktree

Call `worktree_delete` :

```
worktree_delete:
  reason: "Feature merged to main"
```

This will :
1. Run preDelete hooks (e.g., `docker compose down`, `./mvnw clean`)
2. Auto-commit any uncommitted changes (safety net — but you should commit explicitly first)
3. Remove the worktree directory
4. Clean up session state

The branch still exists in git history. The directory is gone.

## Stack-specific notes

### Quarkus (Java + Maven)
- The `worktree.jsonc` for Quarkus projects compiles in `postCreate` to catch issues early
- `target/` is per-worktree (never symlinked) — each worktree compiles independently
- `.m2` is global (`~/.m2`) so dependencies are shared, no re-download
- Use `./mvnw quarkus:dev` in the worktree — hot reload works normally

### Next.js (npm/pnpm)
- `node_modules` is symlinked between worktrees by default (huge time/disk save)
- `postCreate` runs `pnpm install --prefer-offline` to handle minor diffs between branches
- If a worktree has very different deps (major upgrade branch), unlink and reinstall
- `.next/` is excluded (per-worktree build cache)

### Terraform
- **Each worktree has its own `.terraform/`** — never share between worktrees (state corruption)
- `postCreate` runs `terraform init` + `terraform validate`
- **If you `terraform apply` in a worktree, you MUST `terraform destroy` before deleting it**, otherwise you orphan cloud resources
- Use `terraform workspace` if you need multiple environments per worktree

### Kubernetes (manifests in a repo)
- Standard text-file workflow, no special handling needed
- If you have local kind/minikube configs, list them in `copyFiles`

## Multi-worktree coordination

If multiple agents/sessions are working on related features in parallel :

1. **Stay sync'd with `main`.** Periodically rebase your worktree on the latest `main` to avoid drift.
2. **Communicate via PRs/MRs.** When two worktrees touch the same file, the second one to merge handles conflicts.
3. **Don't share state between worktrees.** Each worktree is independent — don't write scripts that assume sibling worktrees exist.

## Anti-patterns

- **Working directly on `main`** for anything non-trivial. Always create a worktree.
- **Long-lived feature worktrees** (multi-week). They drift from `main`. Either rebase regularly or split into smaller features.
- **Forgetting to delete merged worktrees.** They consume disk and clutter `git worktree list`. Run `git worktree prune` periodically.
- **Running `terraform apply` and deleting the worktree without destroy.** You'll leak cloud resources.
- **Using worktrees for one-line fixes.** Overhead > benefit. Just commit on your current branch.

## Quick reference

| Action | Command (via worktree tool) | Command (manual git) |
|---|---|---|
| List worktrees | — | `git worktree list` |
| Create | `worktree_create(branch="feature/x")` | `git worktree add ../x feature/x` |
| Delete | `worktree_delete(reason="merged")` | `git worktree remove ../x` |
| Clean up dead refs | — | `git worktree prune` |

When invoked by an agent : prefer the `worktree_create` / `worktree_delete` tools — they handle terminal spawning, file sync, and hooks automatically.
