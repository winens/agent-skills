---
name: rebase-worktree-conflicts
description: Rebase a worktree branch onto latest main and resolve merge conflicts. Use this skill whenever the user mentions rebasing, conflicts between branches, syncing a worktree with main, or says things like "rebase onto main", "my branch is behind", "merge conflicts", "sync with main", or "update my branch". Also trigger when the user pushes new commits to main and Claude's branch needs to catch up.
---

# Rebase Worktree Branch onto Main

When working in a worktree on a feature branch, the main branch may have advanced with commits that conflict with the feature work. This skill guides the rebase process to bring the feature branch up to date while resolving conflicts cleanly.

## When This Applies

- You're working in a worktree on a feature branch
- Main has new commits that the feature branch doesn't have
- Those new commits may touch the same files you've changed
- The user asks you to sync, rebase, or update the branch

## Step-by-Step Process

### Step 1: Assess the Situation

Before touching anything, understand the current state:

```bash
# What branch are we on?
git branch --show-current

# How far behind main are we?
git fetch origin
git log --oneline HEAD..origin/main
```

Count the commits and note which files they touch:

```bash
git diff --name-only HEAD..origin/main
```

Compare this against your own changed files to anticipate conflicts:

```bash
git diff --name-only origin/main..HEAD
```

The intersection of these two file lists is where conflicts will happen.

### Step 2: Rebase onto Main

```bash
git rebase origin/main
```

This replays each of your commits on top of the latest main, one at a time.

### Step 3: Handle Conflicts

If rebase stops due to conflicts, git will tell you which files are conflicted. For each conflicted file:

1. **Read the file** — look for conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
2. **Understand both sides** — the `HEAD` side is the incoming main commit, the other side is your feature work
3. **Resolve intelligently** — don't just pick one side blindly:
   - If main renamed a function you're using, update your code to use the new name
   - If main restructured a file you're editing, adapt your edits to the new structure
   - If main added imports or fields near your changes, keep both
4. **Stage the resolved file**: `git add <file>`
5. **Continue the rebase**: `git rebase --continue`

Repeat for each conflicted commit until rebase completes.

### Step 4: When Conflicts Are Too Messy

If conflicts are spread across many commits and resolving one-by-one is painful, you can squash your work into a single commit first and then rebase:

```bash
# Abort the current rebase if in progress
git rebase --abort

# Squash all your commits into one
git reset --soft $(git merge-base HEAD origin/main)
git commit -m "feat: <descriptive message for the feature work>"

# Now rebase — only one round of conflicts
git rebase origin/main
```

This trades granular commit history for a much simpler conflict resolution process. Ask the user before squashing — they may prefer to keep individual commits.

### Step 5: Verify

After a successful rebase:

```bash
# Confirm clean state
git status

# Verify your changes are still intact
git log --oneline origin/main..HEAD

# Quick build/test if applicable
```

## Important Rules

- **Never force-push** without explicit user permission. After a rebase, the branch history has changed and requires `git push --force-with-lease`. Always ask before pushing.
- **Never use `--no-verify`** to skip hooks. If a hook fails, investigate and fix the root cause.
- **Preserve user intent** — when resolving conflicts, your feature work should take priority over formatting or style changes from main. But structural changes from main (renames, new parameters, moved files) must be respected.
- **Communicate** — tell the user how many commits were replayed, how many had conflicts, and what you resolved. If a conflict is ambiguous, ask the user rather than guessing.
- **Abort if stuck** — if a conflict is genuinely unclear, `git rebase --abort` is safe and returns to the pre-rebase state. Ask the user for guidance.

## Quick Reference

| Situation | Command |
|---|---|
| Check how far behind | `git log --oneline HEAD..origin/main` |
| Start rebase | `git rebase origin/main` |
| Continue after resolving | `git rebase --continue` |
| Skip a commit (dangerous) | `git rebase --skip` |
| Give up and go back | `git rebase --abort` |
| Squash before rebase | `git reset --soft $(git merge-base HEAD origin/main)` |
| Push after rebase | `git push --force-with-lease` (ask first!) |
