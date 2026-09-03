---
name: rash
description: Write, run, and debug rash (rash-sh/rash) scripts — Ansible-flavoured YAML tasks in one static binary — for on-robot, container-entrypoint, and workstation environment config. Use when editing `.rh` files, running `rash`, translating a shell or Ansible bootstrap into rash, or deciding whether rash fits a config task. Covers the dialect differences from Ansible that bite in practice.
allowed-tools: bash
---

# rash

Declarative local automation: YAML tasks + MiniJinja templates, executed by a single static
Rust binary (`rash`, ~7 MB on Linux, no runtime deps). Syntax is Ansible-*like*, but it is a
local scripting tool: no inventory, no SSH, no plays/roles/facts. Verified against v2.21.0.

Use it for: robot / edge-device environment setup, container entrypoints, CI bootstrap,
dotfiles. Do not use it for: fleet orchestration, anything that needs Ansible module parity.

Docs: <https://rash-sh.github.io/docs/rash/master/> (module pages are `module_<name>.html`).
Source of truth for a module's params is `rash_core/src/modules/<name>.rs` (`struct Params`).
Parameter tables for the modules that matter here: [reference/modules.md](reference/modules.md).
Tested end-to-end example: [examples/robot-bootstrap.rh](examples/robot-bootstrap.rh).

## Install on a target (binary)

Releases ship `rash-<ver>-<triple>.tar.gz` containing a single `rash` file. Statically
linked (needs no glibc at runtime; the gnu build works on Ubuntu 22.04/24.04 and Debian).

| Triple | Use for |
|---|---|
| `x86_64-unknown-linux-gnu` / `-musl` | x86 controllers, containers |
| `aarch64-unknown-linux-gnu` | Jetson, Raspberry Pi, other arm64 Linux (no aarch64 musl build) |
| `aarch64-apple-darwin` | local authoring on a Mac |

```bash
RASH_VERSION=2.21.0
ARCH=$(uname -m)   # x86_64 | aarch64
curl -fsSL "https://github.com/rash-sh/rash/releases/download/v${RASH_VERSION}/rash-${RASH_VERSION}-${ARCH}-unknown-linux-gnu.tar.gz" \
  | sudo tar xz -C /usr/local/bin rash
rash --version
```

Also: `cargo install rash_core` (needs a current stable toolchain; 1.98 builds, 1.95 did not),
`ghcr.io/rash-sh/rash:latest` image, AUR `rash`. Pin the version on robots.

## Script skeleton

```yaml
#!/usr/bin/env -S rash --
#
# One-line purpose.
#
# Usage:
#   setup.rh [options] <robot-id>
#
# Options:
#   --fleet=NAME   fleet name [default: lab]

- name: Validate inputs
  assert:
    that:
      - robot_id is defined
      - env.HOME is defined

- name: Render config
  template:
    src: "{{ rash.dir }}/templates/robot.toml.j2"
    dest: /etc/robot/robot.toml
    mode: "0644"
  notify: restart robot

handlers: []   # only valid in the tasks:/handlers: layout, see below
```

- Always use the `#!/usr/bin/env -S rash --` shebang. With plain `#!/usr/bin/env rash`, the
  script's own options (`--verbose`, `--fleet=x`) are consumed by rash itself.
- Two file layouts: a plain task list, or a mapping with `tasks:` and optional `handlers:`.
  Handlers exist only in the second layout.
- Extension `.rh`, `chmod +x`. Templates are plain `.j2` files rendered with the same context.

## Running

```
rash [OPTIONS] script.rh -- <script args>
  -c/--check  -d/--diff        dry-run; show unified diffs of file changes
  -e KEY=VALUE                 inject into {{ env }}; repeatable
  -o ansible|raw|json          raw = module output only; json = one JSON object per task line
  -v / -vv                     debug / trace (or RASH_LOG_LEVEL=DEBUG|TRACE)
  -b/--become [-u USER]        escalate every task (see Privilege below)
  --script '<yaml>' name.rh    inline script; start it with "#!..." or use --script=… so clap
                               does not choke on a leading "- "
```

Exit code is 0 on success and 1 on any failure: failed task, failed assert, undefined variable,
YAML error, bad script args, and also `--help` (help is printed as `[ERROR]`, rc 1).

Development loop: `rash --check --diff x.rh` → `rash x.rh` → `rash x.rh` again and confirm
every task prints `ok`, not `changed`. The second run is the idempotency test.

## Task keywords (complete list)

| Keyword | Notes |
|---|---|
| `name` | Templated. |
| `<module>` | Exactly one per task. Params are templated before execution. |
| `when` | Expression **without** `{{ }}`; string or list (list = AND). Undefined variable is a fatal error, but any other template error (e.g. a filter applied to `none`) **fails open and runs the task** with nothing logged. Guard with `is defined` / `default()` and see gotcha 15. |
| `loop` | List, or `"{{ expr }}"` yielding a list. Item is `item`. No `loop.index`, no `loop_control`, no `with_*`. |
| `register` | Stores `{changed, output, extra}`. In a loop only the **last** iteration survives. Not set when the task fails, even with `ignore_errors`. |
| `vars` | Task-scoped; later keys may reference earlier ones. Do not leak. |
| `environment` | Map of env vars passed to the spawned process (`command`, `shell`; verified); templated. Does not persist between tasks. |
| `changed_when` | Expression without `{{ }}`; may not reference this task's own `register`. Affects the log line only, not `register.changed`. |
| `ignore_errors` | Boolean literal only (not templated). Swallows module failures; the task's `register` stays unset. |
| `check_mode` | Per-task dry-run. `rash.check_mode` is true under `--check`. |
| `become`, `become_user`, `become_method`, `become_exe`, `become_password` | See Privilege. |
| `notify` | Handler name or list. Fires only when the task reports changed. Unknown handler = warning. |
| `rescue`, `always` | Allowed on any task, not just `block`. `rescue` catches module failures **and** template render errors. |
| `retries`, `delay`, `until` | `until` is an expression over a `register`ed var (you must register). Default 3 retries, delay 0 s. |
| `async`, `poll` | Background `command`/`shell`; `poll: 0` = fire-and-forget, `register` gets `rash_job_id`; check with `async_status: {jid: ...}` (`extra.finished`, `extra.failed`). |

Not in rash: `failed_when`, `tags`, `delegate_to`, `become: yes` on a play, `hosts`, `roles`,
`import_tasks`, `set_fact` (use `set_vars`), `loop_control`, `ansible_*` facts.
Unknown keys or module names produce the misleading error
`Expected a YAML mapping with tasks (and optional handlers), got: Sequence [...]`.

## Templating and variables

MiniJinja (`{{ }}`, `{% %}`), strict undefined. Context:

- `rash.path`, `rash.dir` (absolute script location), `rash.args` (raw script args),
  `rash.user.uid/gid`, `rash.check_mode`.
- `env.NAME` for every environment variable plus `-e` overrides. Missing name is an error;
  write `env.NAME | default('x')` or guard with `'NAME' in env`.
- Docopt results: positionals as lowercase snake_case (`<robot-id>` → `robot_id`, repeatable
  ones are lists), commands as booleans, options under `options.<name>` (missing value → `null`,
  flags → `false`). Options must be passed **before** positionals on the command line, and
  `[options]` must be the first element of the usage pattern.
- Lookups (functions): `file(path)`, `find({'paths': ..., 'patterns': [...]})`,
  `pipe('cmd')` (shell, cwd = script dir), `password(path, length=, chars=, seed=)`,
  `passwordstore(...)`, `vault(...)`. Plus MiniJinja `range`, `dict`, `namespace`.
- `omit`: `mode: "{{ env.MODE | default(omit) }}"` drops the param entirely.
- `debug()` dumps the whole context **including every environment variable** (tokens, passwords).
  Never leave it in a script that logs anywhere; prefer `debug: {var: some.path}`.

Filters available (the full set): `abs attr batch bool capitalize chain count default(d)
dictsort escape(e) first float format groupby indent int items join last length lines list
lower map max min pprint reject rejectattr replace reverse round safe select selectattr slice
sort split string sum title tojson trim unique upper urlencode zip`.
Tests: `defined undefined none true false boolean string number int float integer sequence
mapping iterable odd even divisibleby startingwith endingwith in eq ne lt le gt ge sameas`.

Not available (write around them):

| Wanted | Use instead |
|---|---|
| `basename` | `path | split('/') | last` |
| `dirname` | `path | split('/') | slice(0, -1) | join('/')` or `pipe('dirname ' ~ path)` |
| `regex_replace`, `regex_search` | `replace()` for literals; `pipe('sed ...')` for regex |
| `b64decode/encode`, `to_yaml`, `from_yaml` | `slurp`/`copy` + `pipe('base64 -d ...')`; `tojson` exists |
| `truncate` | `text[:20]` slicing |
| `split()` without an argument | returns an iterator; add `| list` before `length`/indexing |

## Register shapes you will actually use

```yaml
r.changed / r.output / r.extra
command, shell:  output = stdout; extra = {rc, stderr}      # only on success (rc 0)
                 output is **none**, not "", when the command printed nothing
stat:            extra.stat.{exists, isdir, isreg, islnk, mode, uid, gid, size, mtime, checksum, md5}
find:            extra = [ "/abs/path", ... ]               # a plain list of paths
tempfile:        output = created path
get_url:         extra = {dest, url, size, status_code} on download; null when already current
async (poll 0):  register = {changed, rash_job_id}; loop → {rash_job_ids: [...]}
```

Non-zero exit from `command`/`shell` is a task failure. To inspect a failing command's
output, append `|| true` in `shell` and read `output`, or wrap it with `rescue`.

## Privilege escalation

Default `become_method: syscall` calls `setuid/setgid` directly: the rash process must already be
root or have `cap_setuid,cap_setgid` (`sudo setcap cap_setgid,cap_setuid+ep $(which rash)`).
On a robot the simple pattern is: run the whole script as root (`sudo ./setup.rh`) and use
`become: true` + `become_user: robot` only for the tasks that must drop privileges.
`become_method: sudo` re-invokes rash through `sudo -u USER` per task; it works for an
unprivileged operator with NOPASSWD sudo but does not support handlers or rescue/always.

## Module cheat sheet for environment config

All parameters are templated; quote modes (`"0644"`). See reference/modules.md for full tables.

| Need | Module | Notes verified in v2.21.0 |
|---|---|---|
| Run a program | `command` (`cmd`/`argv`, `chdir`, `transfer_pid`) | No shell; whitespace-split, quotes respected. `transfer_pid: true` execs it as the final step (entrypoints). |
| Pipes, redirects, `creates`/`removes` | `shell` (`cmd`, `executable`, `chdir`, `creates`, `removes`, `stdin`) | Default `/bin/sh`. `creates` gives idempotency to one-shot installers. |
| Write a file | `copy` (`content` or `src`, `dest`, `mode`, `dereference`) | Does **not** create parent dirs; `file: state: directory` first. `src` may be a directory (recursive). `mode: preserve`. |
| Render config | `template` (`src`, `dest`, `mode`) | `src` resolves against the **cwd**, so write `"{{ rash.dir }}/x.j2"`. Does not create parent dirs. Undefined variable in the template fails the task. |
| Dirs, perms, delete | `file` (`path`, `state: file/directory/touch/absent`, `mode`) | `state: file` on a missing path is an error; use `touch`. `absent` removes dirs recursively. |
| One line in a file | `lineinfile` (`path`, `regexp`, `line`, `state`) | Creates the file (and parents) if missing. No `insertafter`, `backrefs`, `create`, `owner`. |
| Regex edit | `replace` (`path`, `regexp`, `replace`, `after`, `before`, `backup`, `validate`) | Multiline mode; `\1` backrefs. |
| Key/value config | `ini_file`, `json_file` (`path`, `section`/`key`, `option`, `value`, `state`) | `json_file.key` is dot-notation. |
| Load vars | `setup: {from: [.env, vars.yaml]}` | `from` must be a **list**. `.env` → `env.*`; YAML/JSON → top-level vars. |
| Set vars | `set_vars: {k: v}` | Persists for the rest of the run. Leaks out of `block`, not out of `include`. |
| Reuse tasks | `include: path.rh` | Relative to cwd; use `{{ rash.dir }}`. Task `vars` are visible inside. |
| Packages | `apt` (`name`, `state`, `update_cache`, `cache_valid_time`, `deb`, `purge`) | Idempotent for `state: present`. `state: absent` reports `changed` for already-absent packages with **no apt transaction** (verified against `/var/log/apt/history.log`), so do not gate anything on it. `package` (auto-detect) reported `changed` on an already installed package; prefer `apt`. |
| Services | `systemd` (`name`, `state`, `enabled`, `masked`, `daemon_reload`, `scope`) | Runs `systemctl is-active/is-enabled` first, so it is idempotent. Fails with "No such file or directory" where `systemctl` is absent (containers): guard with a `stat` on `/run/systemd/system`. |
| Users/groups | `user`, `group` | Idempotent **except** `user.group`: passing a primary group re-runs `usermod -g` every time and reports changed (so never `notify` from it). Let `useradd` create the same-named group instead. `groups` + `append: true` adds memberships. |
| Kernel/network | `sysctl` (`sysctl_file`, `reload`), `modprobe` (`persistent`), `hostname`, `timezone`, `mount`, `netplan`, `networkd`, `nmcli`, `route` | `sysctl` writes the file and applies it; it reports changed on every run inside a container where `/proc/sys` cannot be written. |
| Downloads | `get_url` (`checksum`, `force`, `timeout`), `unarchive` (`remote_src`), `git`, `github_release` | `get_url` is idempotent (ok when the file already matches). |
| Inspect | `stat`, `find` (`paths`, `patterns`, `recurse`, `file_type`), `slurp` | `stat` on a missing path is not an error. |
| Flow | `assert: {that: [...]}`, `fail: {msg}`, `debug: {msg|var}`, `block`, `meta: {action: flush_handlers}`, `pause`, `wait_for` (`port`, `host`, `timeout`), `reboot` | `assert` has **no** `msg`; use `fail` + `when` for custom messages. |
| Scheduling | `cron`, `timer` (systemd timers), `at` | |
| Containers | `docker_*`, `podman`, `docker_compose` | |

Handlers run after the last task (or at `meta: flush_handlers`) in **unspecified order**.
If order matters, flush explicitly between them or fold the steps into one handler.

## Gotchas (each verified by running it)

1. `when: "{{ x }}"` is a syntax error; `when: x`. Same for `until` and `changed_when`.
2. Register inside `loop` keeps only the last item (there is no `results` list). Need every
   result? Emit them from one `shell` call, or write per-item marker files and `find` them.
3. A failed task never sets its `register`, so `r.rc` after `ignore_errors` is undefined.
4. `until` without `register` can never be satisfied and fails after the retries.
5. `mode: 600` (bare integer) fails with "expected a string"; `0644` happens to parse as a string
   but quote it anyway.
6. `copy` and `template` into a missing directory fail with "No such file or directory";
   only `lineinfile` creates parents. Put a `file: {state: directory}` task first.
7. `template.src` and `include` paths are cwd-relative. Prefix with `{{ rash.dir }}/`.
8. Handler order is a HashMap order. Do not rely on it.
9. `changed_when` cannot see the task's own `register`. Use `shell` + `creates`, or accept the
   module's own change detection.
10. Script options after positionals (`setup.rh amr-07 --fleet=x`) fail to parse; put options
    first. A trailing `[options]` in the usage pattern breaks parsing of every invocation.
11. `-- --help` exits 1 and prints help as `[ERROR]`; do not treat rc 1 from a help call as a bug.
12. `debug: {msg: "{{ debug() }}"}` prints all env vars, secrets included.
13. `include` with a `set_vars` inside does not export those vars to the parent; `block` does.
14. Module docs on the site sometimes show params that do not exist (`assert.msg`,
    `setup.from` as a string, `b64decode`). Trust `struct Params` in the source.
15. **A `when` whose template errors runs the task.** `register` on a command that
    printed nothing gives `output: none`, so `when: (r.output | trim | length) > 0`
    raises "cannot calculate length of value of type none" — and rash executes the
    task anyway, printing no error. A broken condition means unconditional
    execution, which is the worst possible default for a converging script. Test
    truthiness instead, since `none` is falsy: `when: r.output`.
16. `default()` substitutes for **undefined only**, not for `none` (standard Jinja2,
    but combined with 15 it is an easy trap). `r.output | default('') | trim` still
    fails when `output` is `none`. There is no `default(x, true)` boolean form.
17. A **skipped** task never populates its `register`, and MiniJinja raises on
    attribute access of an undefined name before a filter can apply, so
    `skipped_task.changed | default(false)` is a fatal error. Guard the object:
    `(skipped_task | default({})).changed | default(false)`.
18. `copy` has no `validate` parameter (Ansible's does). Validate a rendered config
    in a separate task before whatever consumes it restarts.

## Patterns

Guard systemd on hosts/containers without it:

```yaml
- stat: {path: /run/systemd/system}
  register: sd
- systemd: {name: robot-agent, enabled: true, state: started, daemon_reload: true}
  when: sd.extra.stat.exists
```

Gate a service restart on real change. `copy` compares content and `register`
exposes `.changed`, which is more reliable here than handlers and works in the
plain task-list layout:

```yaml
- copy: {content: "...", dest: /etc/svc/config.yaml, mode: "0644"}
  register: cfg
- copy: {content: "...", dest: /etc/systemd/system/svc.service.d/10-x.conf, mode: "0644"}
  register: dropin
- systemd: {name: svc, state: restarted}
  when: cfg.changed or dropin.changed
```

Note `changed_when` affects only the log line, not `register.changed`, so it cannot
be used to shape a gate like this one.

Idempotent one-shot installer:

```yaml
- shell:
    cmd: curl -fsSL https://example.com/install.sh | sh
    creates: /opt/vendor/bin/tool
```

Container entrypoint hand-off (must be the last task). The README's `rash.argv` does not
exist, and `argv: "{{ rash.args }}"` renders to a string and fails; join the args:

```yaml
- command:
    cmd: "{{ rash.args | join(' ') }}"
    transfer_pid: true
```

Secrets: pass them as environment (`-e` or the process env), load with `setup: {from: [/etc/robot/.env]}`,
reference as `env.X`; never bake them into `.rh` files or print them with `debug`.

Machine-readable results: `rash -o json x.rh` emits one `{"changed","output","extra"}` line per
task execution; failures still go to stderr as `[ERROR] ...`.

## Debugging checklist

- `rash -vv x.rh` shows every rendered param (`rendering "..."`) and the module invoked.
- "undefined variable 'x' in template: ..." names the exact template; add `| default()` or a
  `when: x is defined`.
- "Expected a YAML mapping with tasks (and optional handlers)" almost always means a typo in a
  module name or a task keyword that does not exist (e.g. `failed_when`).
- "Options must be preceded by `--`" means the shebang lacks `-S rash --`, or an option came
  after a positional argument.
- `[ignoring error] ...` lines come from `ignore_errors`; `[WARNING] Main task execution failed`
  followed by `Executing rescue tasks` is the `rescue` path.
