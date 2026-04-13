---
name: jujutsu
description: "**REQUIRED** - Always activate FIRST on any VCS operations (commit, status, branch, push, etc.), especially when HEAD is detached. If `.jj/` exists -> this is a Jujutsu (jj) repo - git commands will corrupt data. Essential safety instructions inside."
allowed-tools: Bash(jj *)
---

# Jujutsu (jj) Version Control System

This skill helps you work with Jujutsu, a Git-compatible VCS with mutable commits and automatic rebasing.

## Important: Automated/Agent Environment

When running as an agent:

1. **Always use `-m` flags** to provide messages inline rather than relying on editor prompts:

```bash
jj desc -m "message"      # NOT: jj desc
jj squash -m "message"    # NOT: jj squash (which opens editor)
```

2. **Verify operations with `jj st`** after mutations (`squash`, `abandon`, `rebase`, `restore`).

## Core Concepts

### The Working Copy is a Commit

Your working directory is always a commit (referenced as `@`). Changes are automatically snapshotted when you run any jj command. There is no staging area.

### Commits Are Mutable

Unlike git, jj commits can be freely modified:

1. Before starting work, run `jj st`. If `@` already has changes, run `jj new` first. If `@` is empty, use it as-is.
2. Describe your intended changes with `jj desc -m "Message"`
3. Make your changes.
4. Do NOT run `jj new` when finished — leave that to the next task's step 1.

### Change IDs vs Commit IDs

- **Change ID**: A stable identifier (like `tqpwlqmp`) that persists when a commit is rewritten
- **Commit ID**: A content hash (like `3ccf7581`) that changes when commit content changes

Prefer using Change IDs when referencing commits in commands.

## Essential Workflow

### Starting Work: Describe First, Then Code

```bash
jj desc -m "Add user authentication to login endpoint"
# ... edit files ...
jj st
```

### Viewing History

```bash
jj log              # Recent commits (may elide intermediate commits)
jj log -r 'main::@' # All commits between main and working copy (no elision)
jj log -p           # With patches
jj log --stat       # With file change summary
jj show <change-id> # Specific commit
jj diff              # Working copy diff
jj diff -r <rev>    # Diff of a specific revision
jj diff -r <rev> --stat  # File change summary of a specific revision
```

**Note**: The default `jj log` revset elides intermediate commits with `~` markers.
Use `-r 'main::@'` or `-r 'main::bookmark'` to see the full chain without elision.

### Moving Between Commits

```bash
jj new                            # New empty commit on top of current
jj new && jj desc -m "message"   # New commit with message
jj edit <change-id>               # Edit an existing commit
jj prev -e                        # Edit previous commit
jj next -e                        # Edit next commit
```

## Refining Commits

```bash
jj squash              # Move changes from current into parent
jj squash --from A --into B -m "msg"  # Squash specific commit into another
jj absorb              # Auto-distribute changes to ancestor commits
jj abandon <change-id> # Remove a commit (descendants rebased to parent)
jj undo                # Reverse last jj operation
jj restore             # Discard all working copy changes
jj restore path/to/file.txt                    # Discard specific file
jj restore --from <change-id> path/to/file.txt # Restore from revision
```

**Note**: `jj squash -i` and `jj split` are interactive and will hang in agent environments. Avoid them.

### Squashing multiple commits

To squash several commits into one, use `--from` with multiple revisions:

```bash
# Squash commits B, C, D into A (moves their changes into A, abandons them)
jj squash --from 'B | C | D' --into A -m "Combined description"
```

All descendants of the squashed commits are automatically rebased. If the commits
have remote bookmarks, they will be immutable by default — use `--ignore-immutable`
to override.

**IMPORTANT**: Before squashing commits with remote bookmarks, track all affected
bookmarks first (`jj bookmark track <name>@origin`). Otherwise, pushing
later creates divergent change IDs that are painful to clean up.

## Working with Bookmarks (Branches)

Bookmarks are jj's equivalent to git branches:

```bash
jj bookmark create my-feature -r@          # Create at current commit
jj bookmark move my-feature --to <change-id>  # Move to different commit
jj bookmark list                           # List bookmarks
jj bookmark list --tracked                 # List tracked bookmarks only
jj bookmark delete my-feature              # Delete bookmark
jj bookmark track my-feature@origin             # Track a remote bookmark
jj bookmark set my-feature -r <commit-hash>    # Set bookmark to specific commit
```

**IMPORTANT**: Unlike git branches, jj bookmarks do not automatically move when you create new commits. You must manually update them before pushing.

### Tracking remote bookmarks

Tracking links a local bookmark to its remote counterpart. This is essential
before rewriting commits — without tracking, jj doesn't know the local and
remote bookmarks correspond, and later tracking creates divergent change IDs.

```bash
# Track before rewriting
jj bookmark track my-feature@origin

# After rewriting and tracking, push all tracked bookmarks at once
jj git push --tracked
```

## Git Integration

```bash
jj git clone <url>           # Clone a git repository
jj git init --colocate       # Initialize jj in an existing git repo
jj git push -b <bookmark>   # Push a bookmark to remote
jj git fetch                 # Fetch from remote
```

### Colocated Repos (.jj/ and .git/ both exist)

Both jj and git commands work, but:
- ALWAYS ensure work is committed in jj before switching to git
- After git operations, jj detects changes on next command

### Pushing Changes

```bash
# Move bookmark to current commit, then push
jj bookmark move my-feature --to @
jj git push -b my-feature
```

## Handling Conflicts

jj allows committing conflicts — resolve them later. Do not use `jj resolve` (interactive). Edit conflicted files directly, then `jj st` to verify.

## Quick Reference

| Action | Command |
|--------|---------|
| Describe commit | `jj desc -m "message"` |
| View status | `jj st` |
| View log | `jj log` |
| View diff | `jj diff` |
| New commit | `jj new` then `jj desc -m "message"` |
| Edit commit | `jj edit <id>` |
| Squash to parent | `jj squash` |
| Auto-distribute | `jj absorb` |
| Abandon commit | `jj abandon <id>` |
| Undo last operation | `jj undo` |
| Restore files | `jj restore [paths]` |
| Create bookmark | `jj bookmark create <name>` |
| Move bookmark | `jj bookmark move <name> --to @` |
| Push bookmark | `jj git push -b <name>` |
| Track bookmark | `jj bookmark track <name>@origin` |
| Rebase onto | `jj rebase -d <destination>` |

## Best Practices

1. **Describe first**: Set commit message before coding
2. **One change per commit**: Keep commits atomic
3. **Use change IDs**: They're stable across rewrites
4. **Refine commits**: Leverage mutability for clean history
5. **No staging area**: Just make changes, they're automatically tracked
