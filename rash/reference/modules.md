# rash module parameters (v2.21.0)

Extracted from `rash_core/src/modules/<name>.rs` `struct Params`. Every value is templated
before the module runs; most modules receive all values as strings, so quote numbers that are
modes or versions. `Option` fields are optional. Enum values are lowercase in YAML
(`state: absent`).

Full module list (about 200): acl alternatives apk apt apt_hold apt_repository archive assemble
assert async_status async_poll at auditd authorized_key aws_s3 blkdiscard block borgmatic btrfs
cargo certbot cgroups chroot cloud_init cloudflare_dns command composer conntrack consul_kv copy
cron cronvar crypttab dconf debconf debootstrap debug distro_package dmsetup dnf docker_compose
docker_config docker_container docker_exec docker_image docker_info docker_login docker_network
docker_prune docker_volume dpkg_selections elasticsearch ethtool expect fail fail2ban fetch file
filesystem find firewalld flatpak gem get_url git github_release gpg_key grafana
grafana_dashboard group grub haproxy helm helm_info homebrew hostname htpasswd include incus
ini_file initramfs interfaces_file ipaddr iptables iscsi iso_extract java_keystore jenkins_job
json_file kafka_topic kernel_blacklist known_hosts kubectl kubernetes lbu libvirt lineinfile
locale logrotate luks lvg lvm_snapshot lvol lxd_container make mdadm meta modprobe
mongodb_collection mongodb_db mongodb_replicaset mongodb_user mount mqtt mysql_db mysql_query
mysql_replication mysql_user netbox_ipam netplan networkd nftables nginx nmcli npm nsupdate
openrc openssl_certificate openssl_csr openssl_privatekey opkg package pacman pam_limits parted
passwordstore patch pause pids ping pip podman postgresql_db postgresql_query postgresql_user
poweroff prometheus prometheus_rule proxmox rabbitmq_user rclone reboot redis replace restic
route runit script seboolean selinux service set_vars setup sgdisk shell slurp smartctl
ssh_config sshd_config stat sudoers supervisor swapfile synchronize sysctl sysfs syslog systemd
tailscale tempfile template timer timezone trace ufw unarchive uri user vault vault_secret
vault_token vdo wait_for wakeonlan wipefs wireguard xattr xml yum_repository zfs zpool zypper.

Docs page per module: `https://rash-sh.github.io/docs/rash/master/module_<name>.html`.

## Execution

### command
Free-form string (`command: ls -la`) or mapping.
| param | type | notes |
|---|---|---|
| `cmd` | string | whitespace-split, shell quotes honoured, **no shell** |
| `argv` | list | alternative to `cmd` |
| `chdir` | string | |
| `transfer_pid` | bool | exec the command in place of rash (PID 1 hand-off). Must be the last task. |
Result: `output` = stdout, `extra = {rc, stderr}`. Non-zero rc fails the task.

### shell
| param | type | notes |
|---|---|---|
| `cmd` | string | run via `executable -c` |
| `executable` | string | default `/bin/sh` |
| `chdir` | string | |
| `creates` | path | skip when it exists |
| `removes` | path | skip when it does not exist |
| `stdin` | string | |

### script
| `path` | required | `args` string | `chdir` | `executable` (default: shebang) |

### include
`include: path.rh` — path is cwd-relative; use `{{ rash.dir }}`. Task `vars` are visible inside.
`set_vars` inside the include do **not** propagate out.

### block
`block: [tasks...]` with optional `rescue:` / `always:` / `when:` / `vars:` on the same task.
`set_vars` inside a block **do** propagate out; block `vars` do not.

### meta
`meta: {action: flush_handlers}` — run pending handlers now.

### async_status / async_poll
`async_status: {jid: N}` → `extra = {jid, status, finished, failed, output, error, changed}`.
`async_poll: {jid: N, interval: S}` blocks until done.

### pause
`seconds` / `minutes` / `prompt`.

### wait_for
| `port` | u16 required | `host` (default localhost) | `timeout` s | `connect_timeout` s |
Fails on timeout unless `ignore_errors: true`.

## Files

### copy
| param | notes |
|---|---|
| `content` **or** `src` | `src` may be a directory (recursive) |
| `dest` | absolute; parent dir must exist |
| `mode` | `"0644"` or `preserve` |
| `dereference` | default true; false copies symlinks as symlinks |

### template
| `src` | cwd-relative or absolute (use `{{ rash.dir }}`) | `dest` | `mode` (`"0644"` / `preserve`) |
Parent dir must exist. Whole context is available inside the template.

### file
| param | notes |
|---|---|
| `path` | required |
| `state` | `file` (default; errors if missing), `directory` (mkdir -p), `touch`, `absent` (recursive) |
| `mode` | string |
No `owner`/`group` params; use `command: chown`.

### lineinfile
| `path` | `regexp` (Python-style; replaces the matching line, else appends) | `line` (required unless absent) | `state` present/absent |
Creates the file and parent dirs when missing. No `insertafter`, `insertbefore`, `backrefs`,
`create`, `backup`, `owner`.

### replace
| `path` | `regexp` (MULTILINE) | `replace` (default `""`, `\1` backrefs) | `after` | `before` | `backup` | `validate` (`%s` placeholder) | `encoding` |

### ini_file
| `path` | `section` (omit → before first section) | `option` | `value` (required if present) | `state` | `no_extra_spaces` |

### json_file
| `path` | `key` dot-notation | `value` any JSON | `state` | `backup` |

### stat
| `path` | `follow` | `get_checksum` (true) | `checksum_algorithm` md5/sha1/sha256 | `get_md5` | `get_mime` | `get_attributes` |
`extra.stat`: exists isdir isreg islnk isfile isfifo issock isblk ischr mode uid gid pw_name
gr_name size atime mtime ctime inode dev nlink checksum md5 mimetype readable writeable executable.
Missing path → `exists: false`, no error.

### find
| `paths` list required | `patterns` list (basename globs) | `excludes` | `file_type` file/directory/link/any | `recurse` (false) | `hidden` (false) | `follow` | `size` (`+1MB`, `-10KiB`) |
`extra` is a flat list of absolute paths. Also usable as a lookup: `{{ find({'paths': '/x'}) }}`.

### tempfile
| `state` file/directory required | `path` parent dir | `prefix` | `suffix` (files only) | `mode` |
`output` = created path.

### slurp
`src` → `extra.content` base64 (no `b64decode` filter exists; prefer `file()` lookup for text).

### get_url
| `url` | `dest` (file or dir ending in `/`) | `mode` | `owner` | `group` | `checksum` `sha256:...` | `force` | `backup` | `timeout` | `headers` map | `url_username` | `url_password` | `force_basic_auth` | `validate_certs` |
Idempotent: `ok` when the destination already matches.

### unarchive
| `src` (path or URL with `remote_src: true`) | `dest` | `create_dest` | `exclude` list | `mode` | `owner` | `group` | `checksum` |
tar, tar.gz, tar.bz2, tar.xz, zip.

### github_release
| `repo` owner/name | `tag` (`latest`) | `asset` regex | `dest` | `mode` (0755) | `timeout` | `api_token` |

### git
| `repo` | `dest` | `version` **required** (branch/tag/sha) | `depth` | `single_branch` | `update` | `force` | `key_file` | `accept_hostkey` |

### uri
| `url` | `method` | `body` | `headers` | `status_code` list (default [200]) | `timeout` | `return_content` | `url_username` | `url_password` | `force_basic_auth` | `validate_certs` |
`extra = {status, url, headers, content?, json?}`. GitHub's API returns 403 without a
`User-Agent` header.

## Variables and flow

### set_vars
Mapping of new variables; values templated in order.

### setup
`from`: **list** of paths. `.env` → `env.*`; `.yaml`/`.yml`/`.json` → top-level variables;
no extension → sniffed.

### assert
`that`: list of expressions (no `{{ }}`). No `msg` param. Custom message: `fail` + `when`.

### fail
`msg`.

### debug
`msg` (templated string) **or** `var` (expression). `msg: "{{ debug() }}"` dumps the whole
context including all environment variables.

## System

### apt
| param | notes |
|---|---|
| `name` | string or list; `pkg=version` pins |
| `state` | present (default) / absent / latest / build-dep / fixed |
| `update_cache`, `cache_valid_time` (s) | |
| `upgrade` | bool, whole system |
| `deb` | path to .deb |
| `purge`, `install_recommends` (true), `install_suggests`, `default_release`, `allow_downgrade`, `allow_unauthenticated`, `executable` (apt-get), `extra_args` | |
Runs non-interactively. Idempotent.

### package
| `name` | `state` present/absent/latest | `update_cache` | `upgrade` | `use_manager` apk/apt/dnf/pacman/zypper |
Reported `changed` for an already-installed package in testing; prefer the specific module.
Also: `apk`, `dnf`, `pacman`, `zypper`, `opkg`, `apt_repository`, `apt_hold`, `dpkg_selections`,
`debconf`, `pip`, `npm`, `cargo`, `gem`, `flatpak`, `homebrew`.

### systemd
| param | notes |
|---|---|
| `name` | unit |
| `state` | started / stopped / restarted / reloaded |
| `enabled` | bool |
| `masked` | bool |
| `daemon_reload`, `daemon_reexec` | bool, run first |
| `force` | override symlinks |
| `scope` | system (default) / user / global |
Checks `is-active` / `is-enabled` before acting. Needs `systemctl` on PATH and a running
systemd; otherwise "No such file or directory". Also `service` (auto-detects systemd /
sysvinit / openrc; params `name`, `state`, `enabled`, `service_manager`), `openrc`, `runit`,
`supervisor`, `timer`.

### user
| `name` | `state` present/absent | `uid` | `group` | `groups` list | `append` (false: exact set) | `home` | `create_home` (true) | `shell` | `comment` | `system` | `password` (hash) | `remove` (home on absent) |

### group
| `name` | `state` | `gid` | `system` |

### sysctl
| `name` | `value` (string) | `state` | `reload` (true) | `sysctl_file` (default `/etc/sysctl.conf`) | `ignoreerrors` |

### modprobe
| `name` | `params` string | `state` | `persistent` disabled/present/absent (writes `/etc/modules-load.d`, `/etc/modprobe.d`) |

### hostname
| `name` | `use` strategy (auto) |

### timezone
| `name` e.g. `UTC`, `Europe/Berlin` |

### mount
| `path` | `src` | `fstype` | `opts` | `state` mounted / unmounted / remounted / absent |
fstab editing is not implemented yet.

### cron
| `name` (comment marker) | `job` | `state` | `minute` `hour` `day` `month` `weekday` (default `*`) | `special_time` hourly/daily/weekly/monthly/yearly/annually/reboot | `disabled` | `user` | `cron_file` (relative → `/etc/cron.d`) |

### reboot
| `msg` | `delay` s | `check_required` → `extra.reboot_required` | `cancel` | `method` auto/systemctl/reboot/shutdown |

### route
| `destination` (`default` or CIDR) | `gateway` | `interface` | `metric` | `table` | `state` |

### netplan
| `state` | `config` full dict **or** `ethernets`/`bridges`/`bonds`/`vlans`/`wifis` | `renderer` networkd (default) / NetworkManager | `version` 2 | `apply` (true) | `backup` | `directory` `/etc/netplan` | `filename` `01-rash.yaml` |

### networkd
| `name` | `type` network/link/netdev | `state` | `interfaces` | `addresses` | `gateway` | `dns` | `dhcp` | `vlan_id` | `netdev_kind` | `mtu` | `config` raw INI | `directory` | `restart` (true) | `backup` |

### nmcli
| `conn_name` | `state` present/absent/up/down | `conn_type` ethernet/wifi/bridge/... | `ifname` | `ip4` | `gw4` | `dns4` | `autoconnect` | `ssid` | `wifi_sec` |

### docker_container
| `name` | `image` | `state` absent/present/started/stopped/restarted | `env` list | `env_dict` | `ports` | `volumes` | `networks` | `healthcheck` | `memory` | ... |
Also `docker_image`, `docker_network`, `docker_volume`, `docker_compose`, `docker_exec`,
`docker_login`, `docker_prune`, `docker_info`, `podman`.
