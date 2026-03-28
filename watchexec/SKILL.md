---
name: watchexec
description: Watch files for changes and re-run commands automatically — language-agnostic live reload
allowed-tools: bash
---

# watchexec

Watches paths for file changes and re-runs a command. Language-agnostic,
uses efficient OS-level file watching, and respects `.gitignore` by default.

## Syntax

```
watchexec [options] [-- <command>]
```

## Key Options

| Flag | Purpose |
|------|---------|
| `-e exts` | Filter by file extensions (comma-separated) |
| `-w path` | Watch specific directory/file (repeatable) |
| `-i pattern` | Ignore glob pattern (repeatable) |
| `-r` | Restart command on change (for long-running processes like servers) |
| `-c` | Clear screen before each run |
| `-N` | Desktop notification on start/finish |
| `-n` | Run command directly, not in a shell |
| `--stop-signal SIG` | Signal to send when stopping (with `-r`) |
| `--signal SIG` | Send signal to process on change instead of restarting |
| `--fs-events kind` | Only trigger on specific events: `create`, `modify`, `remove`, etc. |
| `--timings` | Print how long commands take |
| `--emit-events-to=json-stdio` | Output JSON events to stdout instead of running commands |

## Examples

### Run tests on Rust file changes
```bash
watchexec -e rs cargo test
```

### Run Flutter tests on Dart changes
```bash
watchexec -e dart -w lib -w test -- flutter test
```

### Restart a server on changes
```bash
watchexec -r -e rs -- cargo run
```

### Ignore build artifacts
```bash
watchexec -i "target/**" -i "build/**" make test
```

### Watch specific files
```bash
watchexec -w Cargo.toml -w Cargo.lock -- cargo build
```

### Run multiple commands (shell mode, default)
```bash
watchexec 'cargo fmt && cargo clippy'
```

### Only trigger on file creation
```bash
watchexec --fs-events create -- echo "new file created"
```

### Stream events as JSON (no command)
```bash
watchexec --emit-events-to=json-stdio --only-emit-events
```

### Show timing information
```bash
watchexec --timings -- cargo test
```

## Environment Variables

When the command runs, watchexec sets these variables with the changed paths:

| Variable | Event |
|----------|-------|
| `$WATCHEXEC_CREATED_PATH` | Files created |
| `$WATCHEXEC_REMOVED_PATH` | Files removed |
| `$WATCHEXEC_RENAMED_PATH` | Files renamed |
| `$WATCHEXEC_WRITTEN_PATH` | Files modified |
| `$WATCHEXEC_META_CHANGED_PATH` | Metadata changed |
| `$WATCHEXEC_COMMON_PATH` | Longest common prefix (prepend to get full paths) |

Multiple paths are separated by `:` on Unix. Disable with `--emit-events=none`.

## Companion Tools

- **checkexec** — conditional rebuild (run once if deps are newer than target); `watchexec` is event-driven (continuous)
- **cargo watch** — Rust-specific wrapper around watchexec with cargo integration
- **fd** — fast file finder; useful for building watch lists
