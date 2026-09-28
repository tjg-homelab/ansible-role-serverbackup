# Ansible Role: serverbackup

[![CI](https://github.com/tjg-homelab/ansible-role-serverbackup/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-serverbackup/actions/workflows/ci.yml)

Scheduled local backups with retention, designed around a **pull** transfer
model: the role produces archives on the host and exposes them to your backup
destination (NAS, backup server) through a dedicated SSH user whose key is
locked to a **read-only rrsync** of the backup tree. The destination fetches
on its own schedule — this host holds **no credentials** to the destination,
so a compromised host cannot touch (or encrypt) your backups.

What it manages:

- **Website backups** — one `tar.gz` per directory under a source root,
  on a systemd timer (weekly by default)
- **MySQL backups** (opt-in) — one gzipped dump per database via
  `mysqldump`, on its own timer (daily by default)
- **Path backups** (opt-in) — one `tar.gz` per configured directory tree
  (e.g. an application's home directory), on a shared timer (daily by default)
- **Retention** — keeps the newest N archives per site/database/path
- **Pull access** — `backup-pull` user + `rrsync -ro` forced command
- **Push transfer** (optional, legacy) — rsync/scp the archives to a remote
  after each run, with optional ProxyJump

## Requirements

- Debian 12/13 or Ubuntu 24.04 (their `rsync` package ships `/usr/bin/rrsync`)
- The `ansible.posix` collection (for `authorized_key`)
- MySQL backups additionally need a local MySQL/MariaDB server and client
  tools (`mysql`, `mysqldump`)

## Role Variables

Highlights — see `defaults/main.yml` for the full annotated list.

| Variable | Default | Description |
|---|---|---|
| `serverbackup_base_dir` | `/var/backups/server-backup` | Root of the backup tree (what the destination pulls) |
| `serverbackup_website_backups_enabled` | `true` | Archive each directory under the source root |
| `serverbackup_website_source_root` | `/var/www/html` | One archive per direct subdirectory |
| `serverbackup_mysql_backups_enabled` | `false` | Opt-in — dump each database (exclusions configurable) |
| `serverbackup_mysql_user` / `_password` | `root` / `""` | Written to a managed `.my.cnf` (vault the password) |
| `serverbackup_path_backups_enabled` | `false` | Opt-in — archive each entry in `serverbackup_path_backups` |
| `serverbackup_path_backups` | `[]` | List of `{name, path, excludes}` — see below |
| `serverbackup_retention_count` | `3` | Newest archives kept per site/database/path |
| `serverbackup_pull_enabled` | `true` | Create the read-only pull user |
| `serverbackup_pull_user` | `backup-pull` | Account the destination connects as |
| `serverbackup_pull_public_keys` | `[]` | **Required when pull is enabled** — the destination's public key(s) |
| `serverbackup_push_enabled` | `false` | Legacy push-to-remote after each run |
| `serverbackup_website_timer_on_calendar` | `weekly` | systemd `OnCalendar` for websites |
| `serverbackup_mysql_timer_on_calendar` | `daily` | systemd `OnCalendar` for MySQL |
| `serverbackup_path_timer_on_calendar` | `daily` | systemd `OnCalendar` for path backups |

### Path backups

For data that is neither a docroot nor MySQL. Each entry produces
`<name>_<date>.tar.gz` under `paths/` in the backup tree:

```yaml
serverbackup_path_backups_enabled: true
serverbackup_path_backups:
  - name: confluence-home
    path: /srv/confluence/home
    excludes:          # optional tar --exclude patterns, matched at any depth
      - temp
      - analytics-logs
```

- `name` is the archive prefix and may only contain `[A-Za-z0-9.-]`. `_`
  is refused because retention matches `<name>_*`, so an entry `app` would
  otherwise prune `app_data` archives.
- All entries run from one service on one timer. A missing source is logged
  and the run continues with the remaining entries, then exits non-zero.
- Live trees are archived as-is. If a file changes while tar reads it, the
  archive is kept and a warning is logged (tar exit 1). Any other tar error
  discards that archive. For a consistent snapshot, stop the application
  first.

### How pull access works

The pull user's `authorized_keys` entry is created as:

```
command="/usr/bin/rrsync -ro /var/backups/server-backup",restrict ssh-ed25519 AAAA...
```

`rrsync -ro` confines the connection to read-only rsync of the backup tree;
`restrict` disables forwarding, PTY, and everything else. The backup tree is
group-owned by the pull user with setgid directories (`2750`) and
group-readable archives (`0640`), so the account can read backups and nothing
else on the host.

On the destination, schedule something like:

```
rsync -rlt backup-pull@host.example.com:websites/ /pool/backups/host/websites/
rsync -rlt backup-pull@host.example.com:mysql/    /pool/backups/host/mysql/
rsync -rlt backup-pull@host.example.com:paths/    /pool/backups/host/paths/
```

(Paths are relative to the rrsync root.)

Use `-rlt` (recurse, preserve symlinks, preserve timestamps), not `-a`. `-a`
implies `-p`, which faithfully reproduces the source-side hardening above
(setgid `2750` dirs, `0640` archives) onto the destination — correct on the
source, where it confines the pull account to the backup tree and nothing
else, but wrong on the destination, where it makes the archives unreadable to
the humans who need to verify them. Let the destination filesystem's own ACL
or ownership policy govern modes there instead.

Enabling pull on a host that already has local backups is safe. setgid only
covers files created after the group change, so the role also re-groups
existing content to the pull user. It then fails the converge if the pull
user cannot read the whole tree, rather than leaving a pull that
authenticates and then cannot open a single file.

### Push mode (legacy)

Set `serverbackup_push_enabled: true` plus the `serverbackup_push_destination_*`
variables and provide an SSH private key (`serverbackup_push_ssh_private_key`,
vaulted). Supports rsync or scp and an optional ProxyJump host. Prefer pull —
push requires storing credentials to the destination on every backed-up host.

## Example Playbook

```yaml
- hosts: webservers
  roles:
    - role: serverbackup
      vars:
        serverbackup_pull_public_keys:
          - "{{ vault_nas_pull_pubkey }}"
        serverbackup_mysql_backups_enabled: true      # hosts running MySQL
        serverbackup_mysql_password: "{{ vault_backup_mysql_password }}"
        serverbackup_retention_count: 4
```

## License

MIT

## Author

rnissen — part of the [tjg-homelab](https://github.com/tjg-homelab) role collection.
