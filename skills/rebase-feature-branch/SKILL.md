---
name: rebase-feature-branch
description: "Rebase a feature branch on its source branch, resolve conflicts, repair Alembic heads when needed, and force-push safely."
---

Rebase the current branch only when it is a feature branch (never `main`/`develop`).

## Inputs

- Optional argument: source branch name (for example: `develop`)
- Optional argument: remote name (default: `origin`)

## Guardrails

1. Ensure this is a git repository.
2. Detect current branch with `git rev-parse --abbrev-ref HEAD`.
3. Abort if current branch is one of: `main`, `master`, `develop`, `dev`, `release`.
4. Refuse detached HEAD.

## Source branch detection

Use this order:

1. Explicit user-provided source branch.
2. PR base branch via `gh pr view --json baseRefName -q .baseRefName` (if available).
3. `develop` (if `<remote>/develop` exists).
4. `main` (if `<remote>/main` exists).
5. `master` (if `<remote>/master` exists).
6. Remote HEAD branch.

## Rebase workflow

1. `git fetch <remote> --prune`
2. If another git operation is in progress, clean it up first:
   - `git rebase --abort || true`
   - `git merge --abort || true`
   - `git cherry-pick --abort || true`
3. If working tree is dirty, auto-stash (`git stash push -u -m "auto-stash before rebase-feature-branch"`) and restore at the end.
4. Run `git rebase <remote>/<source-branch>`.
5. On conflicts, fully resolve all conflicted files:
   - remove conflict markers
   - keep intended logic from both sides
   - stage resolved files
   - run `git rebase --continue`
   - only `git rebase --skip` for truly empty/already-applied commits
6. Repeat until rebase completes and `git status --porcelain` is clean (or only intended unstaged local files from stash pop).

## Fix git issues checklist

Before pushing, ensure:

- no unmerged paths
- no rebase/merge/cherry-pick in progress
- branch still tracks expected remote branch
- no accidental conflict markers remain (`<<<<<<<`, `=======`, `>>>>>>>`)

## Alembic migration re-parenting (backend repos)

If repo contains `src/alembic/versions` and alembic is configured:

1. Run `uv run alembic heads`.
2. If multiple heads exist after rebase, re-parent feature migration(s) on top of the source branch head (linear history).
3. Edit feature migration file(s):
   - update `Revises:` docstring comment
   - update `down_revision` to the latest head from the source branch
4. Do **not** create merge migrations for this case.
5. Re-run `uv run alembic heads` until exactly one head remains.

## Finalization

1. Run relevant fast checks (at least lint/tests affected by conflict resolution).
2. Force-push safely:

```bash
git push --force-with-lease <remote> <current-branch>
```

3. If auto-stash was used, restore it (`git stash pop`) and report any post-pop conflicts.
4. Summarize what was rebased, what conflicts were fixed, and whether Alembic migrations were re-parented.
