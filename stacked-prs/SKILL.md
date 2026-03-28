---
name: stacked-prs
description: Creating and maintaining stacked (dependent) pull requests for large features
allowed-tools: jj, bash, gh
---

# Stacked Pull Requests

Stacked PRs are dependent pull requests where each PR builds on the previous one. Use this pattern for complex features that need logical separation and parallel review.

## When to Use

When developing it is helpful to break large changes into sets of smaller changes that build on top
of each-other to enable easier reviews.
This also works well for agentic development because we can develop a large change in smaller steps.

1. Plan large change by breaking it into stages and document this Plan
2. Develop each change as a separate PR that builds on the work in the previous PRs
3. Push these PRs where the first one depends on main and each later one depends on the change before it.

**DEFAULT: Prefer main-based PRs unless user requests stacking or you detect you are working on a stacked PR**

## Source Control: jj (Jujutsu)

This project uses `jj` (Jujutsu) for version control, not `git`. Repos have a `.jj/` directory.
**Do not use raw git commands** — they can corrupt the jj state. Use `jj git ...` for git operations.

Key differences from git for stacked PRs:
- **Bookmarks** instead of branches: `jj bookmark create`, `jj bookmark move`
- **Automatic rebasing**: jj automatically rebases descendants when you modify a commit
- **No staging area**: changes are tracked automatically
- **Change IDs are stable**: use them to reference commits across rewrites

### Track bookmarks before rewriting

Before any rewrite operation (`squash`, `rebase`, `abandon`) on commits that have
remote bookmarks, **track all affected bookmarks first**:

```bash
# Track all remote bookmarks in the stack
for b in sync-endpoint ai-reading-proxy auth-jwt; do
  jj bookmark track "$b" --remote=origin
done
```

If bookmarks are untracked when you rewrite, jj won't know the local and remote
bookmarks correspond. Later tracking imports the old remote ref, creating divergent
change IDs that cascade across the entire stack. Tracking first avoids this entirely.

After the rewrite, push all tracked bookmarks:
```bash
jj git push --tracked
```

## Bookmark Naming

Use descriptive bookmark names. Sequential numbering is optional — the stack position
is tracked in the PR title suffix and description, not the bookmark name.

## PR Title Convention

Use a `[n]` suffix on the title to show position in the stack. Do NOT include the
total count — it changes when PRs are added or merged.

```
:tada: Add sync endpoint [1]
:tada: Add AI reading proxy [2]
:tada: Add JWT authentication [3]
```

## PR Labels

Apply labels to help reviewers filter and understand the stack:

- `base` — the first PR in the stack (targets `main`)
- `dev` — the final PR in the stack (the one to deploy/test from)
- Content labels based on what files the PR touches (e.g. `rust`, `flutter`, `deployment`)

## Creating Stacked PRs

1. Create Base PR
```bash
# Start from main
jj new main
jj desc -m ":tada: Add sync endpoint"
jj bookmark create sync-endpoint

# Implement base functionality
# ... work ...

# Push and create PR
jj git push -b sync-endpoint
gh pr create --head sync-endpoint --base main --title ":tada: Add sync endpoint [1]" --add-label base
```

2. Create Dependent PR
```bash
# Start from previous bookmark (@ should be on sync-endpoint)
jj new
jj desc -m ":tada: Add AI reading proxy"
jj bookmark create ai-reading-proxy

# Implement dependent functionality
# ... work ...

# Push and create PR
jj git push -b ai-reading-proxy
gh pr create --head ai-reading-proxy --base sync-endpoint --title ":tada: Add AI reading proxy [2]"
# Base: sync-endpoint ← NOT main!
```

## PR Description Template

Every PR in the stack should include a `## PR Stack` section at the bottom of
the description. Bold the current PR's line and add `← this PR`.

```md
## PR Stack

1. PR #23 `sync-endpoint` → `main`
2. **PR #24 `ai-reading-proxy` → `sync-endpoint` ← this PR**
3. PR #38 `auth-jwt` → `ai-reading-proxy`
4. PR #39 `gdpr-export` → `auth-jwt`
5. PR #27 `clear-history` → `gdpr-export`

Merge in order from 1 → 5.
```

When adding a new PR to the stack, update the stack section in all open PRs.

## Landing a PR from the stack

Follow these steps exactly, in order, when merging the bottom PR in the stack.
The ordering is critical — deleting a branch before retargeting will cause GitHub
to auto-close the next PR in the stack.

### Step 1: Merge the PR on GitHub

Merge via the GitHub UI (squash merge or regular merge).

### Step 2: Retarget the next PR (BEFORE deleting the branch)

```bash
gh pr edit <next-pr-number> --base main
```

**WARNING:** If you delete the merged branch before retargeting, GitHub auto-closes
the PR above it. You'll have to recreate it with `gh pr create`. Always retarget first.

### Step 3: Delete the merged bookmark

```bash
jj bookmark delete <merged-bookmark>
jj git push --deleted
```

### Step 4: Fetch and let jj auto-rebase

```bash
jj git fetch
```

jj automatically rebases descendant commits onto updated main. However, auto-rebase
does NOT mean conflict-free: if the merged PR's final content differs from what
descendants were built on (e.g. squash-merge changes), jj will mark those descendants
as `(conflict)`.

### Step 5: Check for conflicts

```bash
jj log -r 'main::deployment-fixes'
# Look for (conflict) markers on descendant commits.
# If conflicts exist, edit the affected commit and resolve:
#   jj edit <change-id>
#   # fix conflicts in the working copy
#   jj squash   # or just let jj snapshot the resolution
```

### Step 6: Verify and push

After resolving conflicts, **you must run the full test suite at each resolved
commit before pushing**. Conflict resolution can produce code that has no conflict
markers but doesn't compile or passes incorrectly — e.g. when a downstream PR
changed function signatures that your resolution still references.

```bash
# Edit into the resolved commit and run tests
jj edit <bookmark>
cargo fmt --check && cargo clippy -- -D warnings && cargo test
# (or flutter test && flutter analyze for Flutter changes)

# Repeat for each commit that had conflicts

# Push all rebased bookmarks
jj git push --tracked

# ⏸ STOP — confirm the PR diffs on GitHub look correct
```

### Step 7: Update PR titles and descriptions

Renumber the `[n]` suffix in each remaining PR title and update the `## PR Stack`
section in every open PR's description. Use `gh pr edit` in a loop.

**IMPORTANT: Work one bookmark at a time when resolving conflicts. Verify the result
and confirm with the user before proceeding to the next. Never batch-rebase the
entire stack autonomously.**

## Rebasing dependent changes after feedback

When a mid-stack PR is changed based on review feedback, jj automatically
rebases all descendants. Commit hashes change but change IDs stay stable.
If your edits touch the same lines as descendant commits, those descendants
will be marked `(conflict)` — resolve them before pushing.

```bash
# After making changes to ai-reading-proxy (PR #24)
# jj has already rebased auth-jwt and everything after it

# Check the stack for conflicts
jj log
# If any descendant shows (conflict), edit it and resolve

# CRITICAL: After resolving each conflict, run the full test suite at that commit.
# Conflict resolution can silently produce broken code (e.g. wrong function
# signatures, missing variables) even when all markers are removed.
jj edit auth-jwt
cargo fmt --check && cargo clippy -- -D warnings && cargo test

# Only push after tests pass
jj git push -b auth-jwt
# ⏸ STOP — verify the PR diff on GitHub before continuing

jj edit gdpr-export
cargo test
jj git push -b gdpr-export
# Continue pattern for each resolved commit
```

## Pushing rebased bookmarks to GitHub

If bookmarks were tracked before the rewrite (see "Track bookmarks before rewriting"),
push all at once:

```bash
jj git push --tracked
```

Otherwise, push each bookmark individually:

```bash
jj git push -b <bookmark-name>
```

If jj reports a non-tracking remote bookmark exists, track it first, resolve the
bookmark to the correct commit, then push:

```bash
jj bookmark track <name> --remote=origin
jj bookmark set <name> -r <correct-commit-hash> --allow-backwards
jj git push -b <name>
```

After pushing, clean up any leftover divergent commits:
```bash
jj abandon --ignore-immutable 'divergent() & ~ancestors(bookmarks())'
```

## Reopening closed/merged PRs

GitHub will not reopen a PR if the head branch has "no new commits" relative to
when it was closed — even after a force-push with rebased content. In this case,
create a new PR with `gh pr create` using the same bookmark, title, and description.

## Updating the stack after structural changes

When PRs are added, removed, merged, or renumbered:

1. Update the `[n]` suffix in each PR title
2. Update the `## PR Stack` section in every open PR's description
3. Move the `base` label if the bottom PR changes
4. Move the `dev` label if the top PR changes

Use a shell loop with `gh pr edit` to update titles and descriptions in bulk.
