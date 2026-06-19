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

Use a `git worktree add` command (opencode will prompt — `git worktree add *` is set to `ask`) :

```bash
git worktree add ../user-csv-export -b feature/user-csv-export      # branches off current HEAD
# or off an explicit base:
git worktree add ../user-csv-export -b feature/user-csv-export main
```

This creates a sibling directory (`../user-csv-export`) on a new branch. Then do the per-stack
setup manually (see "Stack-specific notes" below) — e.g. `pnpm install`, `./mvnw compile`, or
`terraform init` — and copy any needed env files. Tell the user the path; they can `cd` into it
or keep working from the current session (it's the same isolated directory either way).

> Note: there is no `worktree_create` tool in this setup — drive worktrees with plain `git`
> commands, which the bash permissions already allow.

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

First commit or stash anything you want to keep, then run any per-stack teardown manually
(e.g. `docker compose down`, `./mvnw clean`, `terraform destroy` if you applied). Then remove it
(opencode prompts — `git worktree remove *` is `ask`) :

```bash
git worktree remove ../user-csv-export        # add --force if it refuses on dirty state
```

The branch still exists in git history; the directory is gone. Clean up stale refs later with
`git worktree prune`.

> Note: there is no `worktree_delete` tool — `git worktree remove` does the same job. It does
> **not** auto-commit for you, so commit deliberately before removing.

## Stack-specific notes

These are **manual** steps to run in the new worktree after `git worktree add` — there's no
automated post-create hook in this setup.

### Quarkus (Java + Maven)
- Run `./mvnw compile` once after creating, to catch issues early
- `target/` is per-worktree — each worktree compiles independently
- `.m2` is global (`~/.m2`) so dependencies are shared, no re-download
- Use `./mvnw quarkus:dev` in the worktree — hot reload works normally

### Next.js (npm/pnpm)
- Run `pnpm install --prefer-offline` after creating to handle dep diffs between branches
- To save time/disk you *can* symlink `node_modules` from the main worktree
  (`ln -s ../<main>/node_modules node_modules`) — but reinstall fully on a major-upgrade branch
- `.next/` is a per-worktree build cache; don't share it

### Terraform
- **Each worktree must have its own `.terraform/`** — never share between worktrees (state corruption)
- Run `terraform init` + `terraform validate` after creating
- **If you `terraform apply` in a worktree, you MUST `terraform destroy` before removing it**, otherwise you orphan cloud resources
- Use `terraform workspace` if you need multiple environments per worktree

### Kubernetes (manifests in a repo)
- Standard text-file workflow, no special handling needed
- Copy any local kind/minikube configs into the worktree manually if a command needs them

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

| Action | Command |
|---|---|
| List worktrees | `git worktree list` |
| Create | `git worktree add ../x -b feature/x` |
| Delete | `git worktree remove ../x` (add `--force` if dirty) |
| Clean up dead refs | `git worktree prune` |

Drive worktrees with plain `git` commands — they're already covered by the bash permissions
(`git worktree add/remove/prune` are `ask`, `git worktree list` is `allow`). There is no
worktree plugin/tool installed; file sync and per-stack setup are manual (see notes above).
