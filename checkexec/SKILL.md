---
name: checkexec
description: Conditionally execute commands only when dependency files are newer than the target, like make but standalone
allowed-tools: bash
---

# checkexec

`checkexec` conditionally runs a command only if any dependency file is newer
than the target file. It provides `make`-style conditional rebuilds as a
standalone tool.

## Syntax

```
checkexec <target> <dependencies...> -- <command>
```

The `--` separator is required.

## Examples

### Basic: rebuild only if source changed
```bash
checkexec build/my-bin src/my-program.c -- cc -o build/my-bin src/my-program.c
```

### Infer dependencies from command arguments
```bash
checkexec build/my-bin --infer -- cc -o build/my-bin src/my-program.c
```

### With fd to glob dependencies
```bash
checkexec target/debug/hello $(fd -e rs . src) -- cargo build
```

### Flutter web build (skip if no Dart changes)
```bash
checkexec build/web/main.dart.js $(fd -e dart . lib) -- flutter build web
```

### Sass/image processing (skip expensive rebuilds)
```bash
checkexec build/styles.css src/styles.scss -- sass src/styles.scss build/styles.css
```

## Exit Codes

- `0` — target is up to date, command was NOT run
- `1` — a dependency or the command was not found
- Otherwise — passthrough of the command's exit code

## Limitations

- `checkexec` executes commands directly, NOT through a shell
- Shell constructs (`&&`, `||`, pipes) are not supported in `<command>`
- To use shell features, wrap with: `checkexec <target> <deps> -- /bin/bash -c "cmd1 && cmd2"`
- For complex cases, prefer two separate `checkexec` invocations over shell wrapping

## Companion Tools

- **watchexec** — live rebuild on file change (event-driven); `checkexec` is callable (build-step/CI)
- **just** — modern command runner (replaces `make` recipes); `checkexec` adds back conditional rebuilds
- **fd** — fast file finder; great for building dependency lists with glob patterns
