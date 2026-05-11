---
name: nushell
description: Use this skill when working with the Nushell shell (`nu` binary, `.nu` scripts, `config.nu`, `env.nu`). Triggers include translating bash to nu, writing or debugging custom commands, editing nu config, working with structured data pipelines, or any "how do I do X in nushell" question. Nushell's core idea is that commands return structured data (records, lists, tables) instead of text — every other detail flows from that.
---

# Nushell quick reference

The user (Tyler) runs Ghostty with `command = /opt/homebrew/bin/nu` (configured at `~/Library/Application Support/com.mitchellh.ghostty/config.ghostty`). Login shell may still be zsh; only Ghostty launches nu. Config lives at `~/Library/Application Support/nushell/config.nu` and `env.nu`. Open with `config nu` / `config env`.

## Mental model — the only thing that matters

Nushell pipelines pass **structured data** (records, lists, tables, filesizes, datetimes) — not text. That is the whole reason it exists. Two consequences that bite people coming from bash:

1. `ls` returns a table you can `where size > 1mb`, `sort-by modified`, `select name size`. There's no `awk`/`cut`/`sed` because columns are first-class.
2. When you pipe **to** an external command, nu serializes the table to text first. To control the serialization, do `... | get name | to text | ^grep foo` — otherwise you'll pipe an ASCII-art bordered table into grep.

## Bash → nushell cheat sheet

| bash | nushell |
|---|---|
| `$VAR` | `$env.VAR` (env) or `$var` (local) |
| `export X=1` | `$env.X = 1` |
| `${X:-default}` | `$env.X? \| default "default"` |
| `$?` | `$env.LAST_EXIT_CODE` |
| `$1`, `$@` | `def main [first: string ...rest] { ... }` |
| `$(cmd)` | `(cmd)` inline, or `$"...(cmd)..."` in strings |
| `cmd > file` | `cmd o> file` (also `out>`) |
| `cmd >> file` | `cmd o>> file` |
| `cmd 2>&1 \| pager` | `cmd o+e>\| pager` |
| `cmd1 && cmd2` | `cmd1; cmd2` (nu short-circuits on error by default) |
| `find . -name '*.rs'` | `ls **/*.rs` (glob is built in) |
| `grep foo file` | `open file \| lines \| where $it =~ 'foo'` |
| `cat file` | `open --raw file` (without `--raw`, nu auto-parses by extension) |
| `wc -l file` | `open file \| lines \| length` |
| `which cmd` | `which cmd` (returns a table) |
| `for f in *.md; do ...; done` | `ls *.md \| each { \|f\| ... }` |
| `xargs -I{} cmd {}` | `each { \|x\| cmd $x }` or `par-each` for parallel |
| `head -n 5` / `tail -n 5` | `first 5` / `last 5` |
| `sort` | `sort` (lists) or `sort-by col` (tables) |
| `uniq` | `uniq` (also `uniq-by col`) |
| Force external over builtin | prefix with `^` — e.g. `^ls`, `^grep` |

Aliases can do simple substitution but **cannot contain pipes**. For anything with `|`, use `def`.

## Data types — literal syntax

```nu
42  0xff  0o77  0b101            # int
3.14                             # float
"str"  'str'  `str`              # string (double supports interpolation/escapes)
$"hello ($name)"                 # string interpolation; \( escapes literal paren
r#'raw 'string' here'#           # raw string
true  false  null                # bool, nothing
2025-01-15  2min  3hr  1.5day    # datetime, duration
64mb  1.5GiB                     # filesize (math-aware: 1mb + 1kb works)
1..5  0..<5  2..20..2            # ranges (inclusive, exclusive, with stride)
[1 2 3]  [a, b, c]               # list (whitespace OR comma)
{name: "x", age: 30}             # record
[{x:1 y:2} {x:3 y:4}]            # table = list of records
{|x| $x + 1}                     # closure
```

## Variables

```nu
let x = 5            # immutable (default — prefer this)
mut y = 0            # mutable; supports +=, -=, ++=
const PI = 3.14      # parse-time constant; needed for `use`/`source` paths
```

**Closures cannot capture `mut` vars.** This is the #1 trap. If you want to accumulate inside `each`, use `reduce` instead, or rewrite as a `for` loop (which is a statement, not an expression).

```nu
# WRONG
mut total = 0
[1 2 3] | each {|n| $total = $total + $n }   # error

# RIGHT
[1 2 3] | reduce -f 0 {|n, acc| $acc + $n }
```

## Pipelines and `$in`

`$in` is the pipeline input. Use it when a command takes a positional arg but you want the piped value:

```nu
ls | sort-by size | first | get name | cp $in ~/Desktop/
```

Inside a closure, `$in` is the closure's input. Inside a pipeline element, `$in` collects the previous step into a single value before use.

## Tables — the bread and butter

```nu
ls | where size > 1mb | sort-by modified -r | first 10
ps | where cpu > 5 | select pid name cpu
ls | group-by type | transpose key value
ls | update size {|r| $r.size / 1kb}
ls | insert kind {|r| if $r.type == 'dir' { 'folder' } else { 'file' }}
ls | reject readonly num_links inode
ls | rename file kind bytes mod
ls | each {|r| $"($r.name): ($r.size)" }       # row → string
ls | par-each {|r| heavy-op $r }                # parallel each
$t1 | append $t2                                 # rows
$t1 | merge $t2                                  # columns
```

`get` extracts a column as a list; `select` keeps a sub-table:

```nu
ls | get name             # list of strings
ls | select name size     # smaller table
```

Cell paths drill into nested structures: `$rec.users.0.email`. Append `?` for optional: `$rec.maybe_field?`.

## Strings

```nu
"foo bar baz" | split row " "                   # → list
"foo bar baz" | str contains "bar"              # → true
"  hi  " | str trim
"hello" | str upcase                            # also downcase, capitalize, reverse
"foo123" | str replace -r '\d+' 'NUM'           # -r enables regex
$"path is ($env.HOME)/bin"                      # interpolation
"foobarbaz" =~ "bar"                            # true (regex/contains operator)
```

## Custom commands

```nu
# Greet a user. Comments above `def` become help text.
def greet [
  name: string                    # required positional
  --loud (-l)                     # boolean flag with -l shortcut
  --times: int = 1                # flag with default
]: nothing -> string {            # input/output type signature
  let msg = $"Hello, ($name)!"
  let final = if $loud { $msg | str upcase } else { $msg }
  1..$times | each { $final } | str join "\n"
}

# Mutate caller env (cd, $env.X = ...) — needs --env
def --env cdsrc [] { cd ~/src }

# Scripts: define a `main` to receive CLI args
#!/usr/bin/env nu
def main [path: string, --verbose] { ... }
```

## Loading data

`open` auto-parses by extension: JSON, YAML, TOML, CSV, XML, SQLite, NUON. Returns structured data directly — no `jq` needed.

```nu
open package.json | get dependencies | columns
open data.csv | where age > 30 | save filtered.csv
http get https://api.github.com/users/anthropics | get blog
open Cargo.toml | update package.version "1.0.1" | save Cargo.toml
```

`from json` / `to yaml` / etc. when you have a string in hand instead of a file.

## Control flow

```nu
if $x > 0 { 'pos' } else if $x == 0 { 'zero' } else { 'neg' }

match $val {
  1 => 'one',
  2 | 3 => 'two-or-three',
  {name: $n} => $"record with ($n)",
  [_, _, ..$rest] => $"list rest: ($rest)",
  _ => 'other'
}

for x in [1 2 3] { print $x }                # statement; doesn't return value
[1 2 3] | each {|x| $x * 2 }                 # expression; prefer this

try { risky } catch {|err| print $err.msg }
```

## Environment

```nu
$env.PATH                                       # always a list (not a colon string)
$env.PATH = ($env.PATH | prepend '/opt/foo/bin')
$env.PATH = ($env.PATH | append '/opt/bar/bin' | uniq)
hide-env FOO                                    # unset (scoped)
with-env { FOO: BAR } { run-cmd }               # temporary

$env.config.show_banner = false                 # mutate, don't replace $env.config
$env.config.hooks.pre_prompt = [{ ... }]
```

`$nu.config-path`, `$nu.env-path`, `$nu.default-config-dir` tell you where config lives.

## Common gotchas

- **Spaces matter in commands**: `ls-l` is one identifier; you want `ls -l`.
- **`if` is an expression**, but `for`/`while`/`loop` are statements. Use `each`/`reduce`/`generate` to get a value out of iteration.
- **Comparing types**: `1 == "1"` is `false`. Use `into int` / `into string` to coerce.
- **`$env` is case-insensitive**, so `$env.PATH` == `$env.Path`.
- **External command output is bytes** until decoded; `| decode utf-8` if needed.
- **`save` needs `--force` to overwrite** an existing file.
- **`null` short-circuits `?`**: `$x?.y` returns `null` if `$x` is null, no error.
- **Pipes in aliases don't work** — switch to `def`.
- **`exit 1` for script exit codes**; the last expression's value is *not* the exit code.

## Useful built-ins to remember

`describe` (type of value), `explore` (interactive table viewer), `inspect` (pipe-through debug print), `tee` (split a stream), `complete` (run external + capture stdout/stderr/exit), `print` vs returning (return goes through `display_output`, `print` writes immediately), `do` (execute a closure), `with-env`, `into ...` (type conversions), `format` (string formatting), `where`, `find` (search a stream).

## Config snippet starter (`config.nu`)

```nu
$env.config.show_banner = false
$env.config.edit_mode = 'emacs'         # or 'vi'
$env.config.history.max_size = 100_000
$env.config.history.file_format = 'sqlite'
$env.config.completions.case_sensitive = false
$env.config.completions.algorithm = 'fuzzy'

# A few aliases
alias ll = ls -l
alias la = ls -la
alias gs = git status
alias gd = git diff

# Custom commands needing pipes
def gco [branch: string] { git checkout $branch }
def jqp [...args] { ^jq $args | from json }
```

## When in doubt

- `help <command>` and `<command> --help` — both work, output is a table.
- `scope commands | where name =~ 'some'` — search loaded commands.
- `which <cmd>` — shows whether it's a built-in, alias, def, or external.
- Docs: <https://www.nushell.sh/book/>, command reference: <https://www.nushell.sh/commands/>.
