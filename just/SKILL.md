---
name: just
description: Command runner with make-inspired syntax — use for project-specific recipes in justfiles
allowed-tools: bash
---

# just

`just` is a command runner. Recipes are defined in a `justfile` with
make-inspired syntax. Unlike `make`, every recipe is "phony" — `just` runs
commands, not build rules.

`just` searches for `justfile` (case-insensitive, or `.justfile`) in the
current directory and upwards, so it works from any subdirectory.

## Justfile Syntax

```just
# Comments start with #
variable := 'value'

# Recipe with dependencies
build: lint test
  cargo build --release

# Recipe with parameters
deploy target='staging':
  ./deploy.sh {{target}}

# Variadic parameters (+ = one or more, * = zero or more)
test *FLAGS:
  cargo test {{FLAGS}}

# Shebang recipe (runs as a script, not line-by-line)
check:
  #!/usr/bin/env bash
  set -euxo pipefail
  cargo fmt --check
  cargo clippy -- -D warnings

# Suppress command echo with @
@quiet-recipe:
  echo "only output, no echo of command"

# Continue on error with -
clean:
  -rm -rf build/
  -rm -rf target/

# Default recipe (runs when you type just with no args)
default:
  @just --list
```

## Key Features

### Settings

```just
set dotenv-load           # Load .env file
set export                # Export all variables as env vars
set positional-arguments  # Pass args as $1, $2, etc.
set shell := ["bash", "-uc"]  # Change default shell
set quiet                 # Don't echo recipe lines
set lazy                  # Skip evaluating unused variables
```

### Expressions and Substitutions

```just
# String concatenation
full := 'hello' + ' ' + 'world'

# Path joining
config := home_directory() / '.config' / 'myapp'

# Backtick command evaluation
git_hash := `git rev-parse HEAD`

# Conditional expressions
mode := if env('CI', '') != '' { 'release' } else { 'debug' }

# Shell function (run command, capture output)
kernel := shell('uname -r')
```

### Dependencies

```just
# Prior dependencies (run before)
test: build
  ./run-tests

# Subsequent dependencies (run after)
deploy: build && notify
  ./deploy.sh

# Dependencies with arguments
default: (test "unit") (test "integration")

# Parallel dependencies
[parallel]
all: build test lint
```

### Recipe Attributes

| Attribute | Effect |
|-----------|--------|
| `[no-cd]` | Don't change to justfile directory |
| `[private]` | Hide from `just --list` |
| `[confirm]` | Prompt before running |
| `[group('name')]` | Group in `--list` output |
| `[linux]` / `[macos]` / `[windows]` / `[unix]` | OS-conditional |
| `[script]` | Run as script (like shebang) |
| `[doc('text')]` | Custom documentation comment |
| `[parallel]` | Run dependencies in parallel |

### Useful Built-in Functions

| Function | Purpose |
|----------|---------|
| `arch()` | CPU architecture |
| `os()` | Operating system |
| `num_cpus()` | Logical CPU count |
| `env('KEY')` / `env('KEY', 'default')` | Environment variable |
| `require('cmd')` | Find executable or error |
| `justfile_directory()` | Directory containing justfile |
| `invocation_directory()` | Directory where `just` was called |
| `uuid()` | Random UUID v4 |
| `sha256(s)` / `blake3(s)` | Hash strings |
| `datetime('%Y-%m-%d')` | Current date/time |
| `quote(s)` | Shell-escape a string |
| `replace(s, from, to)` | String replacement |
| `trim(s)` | Strip whitespace |
| `path_exists(path)` | Check if path exists |
| `absolute_path(path)` | Resolve relative path |
| `error('message')` | Abort with error |

### Modules and Imports

```just
# Import another justfile (recipes merged into this one)
import 'ci.just'

# Module (recipes namespaced as subcommands)
mod deploy

# Invoke: just deploy build
# Or:     just deploy::build
```

### Variables from Command Line

```just
os := "linux"
build:
  ./build {{os}}
```

```console
$ just os=freebsd build
```

### Exporting Variables

```just
export RUST_BACKTRACE := "1"

# Or per-parameter
test $RUST_LOG="info":
  cargo test
```

## CLI Usage

| Command | Purpose |
|---------|---------|
| `just` | Run default recipe |
| `just recipe args...` | Run recipe with arguments |
| `just --list` | List available recipes |
| `just --summary` | Concise recipe list |
| `just --show recipe` | Print recipe source |
| `just --evaluate` | Print all variable values |
| `just --choose` | Interactive recipe picker (uses fzf) |
| `just --fmt --unstable` | Auto-format justfile |
| `just --dump` | Print formatted justfile |
| `just --dump --dump-format json` | Print justfile as JSON |
| `just --groups` | List recipe groups |
| `just -g` | Run from global justfile |

## Companion Tools

- **checkexec** — conditional execution (only run if deps newer than target)
- **watchexec** — re-run on file changes: `watchexec just test`
- **fd** — fast file finder for building file lists
