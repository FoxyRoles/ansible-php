# ansible-php

Installs PHP from [packages.sury.org/php](https://packages.sury.org/php/)
and configures CLI, FPM pools and log rotation.

## Requirements

- Ansible Core 2.15 or newer (`ansible.builtin.deb822_repository`); checked with
  Ansible Core 2.21.5 and 2.21.6.
- Ubuntu 22.04 (jammy), 24.04 (noble) or 26.04 (resolute), as published in the
  [repository](https://packages.sury.org/php/dists/).
- Gathered facts, root privileges (`become: true`) and the system Python 3
  interpreter on the target.
- The users, groups and home directories used by FPM pools must already exist.

The role follows the [upstream installation instructions](https://packages.sury.org/php/README.txt),
using `deb822_repository` for the source definition. It installs
`ca-certificates`, `python3-debian` and `xz-utils`, then installs the official
`debsuryorg-archive-keyring` package from
`https://packages.sury.org/debsuryorg-archive-keyring.deb`. The package manages
`/usr/share/keyrings/debsuryorg-archive-keyring.gpg`; the old Launchpad signing
key is not used for this repository.

The repository is written to `/etc/apt/sources.list.d/debsuryorg-php.sources`, with
the host's release codename and `Signed-By` pointing to the packaged keyring.
The APT cache is refreshed when the source, keyring package or legacy definitions
change. The role does not use `apt_repository` or `apt_key`.

### Migration from the Launchpad PPA

The migration is described in upstream issues
[#91](https://codeberg.org/oerdnj/deb.sury.org/issues/91) and
[#73](https://codeberg.org/oerdnj/deb.sury.org/issues/73).
Before the first APT operation, the role removes legacy PHP definitions named
`php.list`, `php.sources`, `sury-php.sources`, `ondrej-php.list`, `ondrej-php.sources`,
`ondrej-ubuntu-php-*.list` and `ondrej-ubuntu-php-*.sources` from
`/etc/apt/sources.list.d`. This also handles definitions left by an Ubuntu
upgrade. Remove any custom PPA entries in other files before running the role.

Installed PHP packages are retained. The role does not force downgrades or a
system-wide upgrade to change their origin. PPA and Sury version strings can
sort differently; normal package updates can move them to Sury as newer
versions become available. Check that the required extensions are available
for your release and architecture.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `php_version` | Required | PHP major/minor version as a quoted string, e.g. `"8.4"`. |
| `php_modules` | Required | Package suffixes such as `cli`, `fpm`, `mysql`; installs `php<version>-<suffix>`. |
| `php_default_user` | `web` | Default FPM pool user and group. |
| `php_pools` | A `web` pool; see [defaults/main.yml](defaults/main.yml) | Dictionary of FPM pool definitions. Only used when `fpm` is in `php_modules`. |

The role installs the requested CLI/FPM runtime and `php<version>-common` first.
It then reads compiled-in extensions from that version's `php -n -m` and/or
`php-fpm -n -m` output and the `Provides` metadata of the installed runtime/common
packages. Requested modules covered by either source do not need a separate
package. This uses the actual installed build, including Sury's packaging,
rather than a fixed list or PHP version thresholds. Other module names are
passed to APT as usual; missing packages and misspellings still fail instead
of being silently skipped.
When both CLI and FPM are requested, a compiled-in extension must appear in both
to be skipped. For example, `pcntl` can be compiled into CLI but absent from FPM.

To inspect the complete lists on a host (adjust the version as needed):

```sh
php8.5 -n -m
php-fpm8.5 -n -m
dpkg-query -W -f='${Package}: ${Provides}\n' php8.5-common php8.5-cli php8.5-fpm
```

Each pool accepts `user` (defaults to `php_default_user`), `home` (defaults to
`/home/<user>`), `port` (defaults to `90` plus the version without a dot, e.g.
`9084` for PHP 8.4), `max_children` (40), `start_servers` (5),
`min_spare_servers` (3) and `max_spare_servers` (5). Set distinct ports when
configuring multiple pools for the same version. Pools listen on `127.0.0.1`.
The package's `www.conf` is replaced with a comment-only file unless you define
your own `www` pool. This disables the default pool while retaining a locally
modified `dpkg` conffile, which normal upgrades preserve with the default
`--force-confdef --force-confold` policy. Explicitly forcing new conffiles can
overwrite local configuration. When `php_pools` contains `www`, the role renders
its pool definition into `www.conf` instead.

## PHP configuration and logs

[files/user_config.ini](files/user_config.ini) is copied to
`/etc/php/user_config.ini` and linked as `99-user_config.ini` in the selected
version's CLI and/or FPM `conf.d` directory. All versions configured by this
role share these settings. Edit that file to change them; `php_ini_settings`
is no longer used.

The supplied settings disable `expose_php`, `.user.ini` files and displayed
errors, set upload and POST limits to 128M, use the `Europe/Prague` timezone,
and set session lifetimes to 30 days. They suppress notices and deprecations
in PHP error reporting. PHP 8.4 deprecated `E_STRICT`, so the configuration
does not reference it.

When migrating from the old `php_ini_settings` implementation, move any needed
custom settings to `user_config.ini`. Existing edits in the package's `php.ini`
are retained; settings in `conf.d` take precedence when they define the same key.

FPM error and slow logs are written to `/var/log/php/<version>-fpm.log` and
`/var/log/php/<version>-fpm.slow.log`. Logs rotate daily with 14 rotations,
compression and a signal to reopen the logs. Old logs in `/var/log` are retained.
FPM services are enabled and started; configuration changes notify the matching
version's restart handler. Shared INI changes notify all FPM instances declared
through this role in the play. CLI-only installations skip FPM configuration,
logging and service management.

## Example playbook

```yaml
---
- name: Configure PHP
  hosts: myserver
  become: true
  roles:
    - role: sunfoxcz.php
      php_version: "8.4"
      php_modules:
        - bcmath
        - bz2
        - cli
        - curl
        - fpm
        - gd
        - intl
        - mbstring
        - mysql
        - opcache
        - readline
        - soap
        - sqlite3
        - xml
        - zip
      php_pools:
        myapp:
          user: myapp
          home: /www/myapp
          port: 9001
          max_children: 120
          start_servers: 20
          min_spare_servers: 10
          max_spare_servers: 20
    - role: sunfoxcz.php
      php_version: "8.3"
      php_modules:
        - bcmath
        - bz2
        - cli
        - curl
        - fpm
        - gd
        - intl
        - mbstring
        - mysql
        - opcache
        - readline
        - soap
        - sqlite3
        - xml
        - zip
      php_pools:
        legacyapp:
          user: myapp
          home: /www/myapp
          port: 9000
          max_children: 120
          start_servers: 20
          min_spare_servers: 10
          max_spare_servers: 20
```

Choose extensions available for the selected PHP version. Existing module lists
may include bundled modules such as `json`, `opcache`, `ctype` or `fileinfo`;
the role detects whether they are supplied by the selected PHP build and packages.
For CLI only, set `php_modules` to
`[cli]` (plus any required extensions) and omit `php_pools`.

## License

Licensed under MIT license. See [LICENSE](LICENSE.md) for details.
