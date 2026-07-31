# aux4/db-mysql-backup

Backup and restore provider commands for MySQL. This package extends the
`db:mysql` profile (owned by `aux4/db-mysql`) with `backup` and `restore`
commands that wrap the `mysqldump` and `mysql` system CLIs.

It is packaged **separately** from `aux4/db-mysql` on purpose: the core
`aux4/db-mysql` query commands (`execute`, `stream`, `describe`, `list`) use the
Node driver and need no external binaries. Only backup/restore need the
`mysql-client` CLIs, so that system dependency is isolated here — install this
package only when you need dump/restore.

Because it declares the same `db:mysql` profile, its commands merge into the
existing profile: after installing both packages, `aux4 db mysql --help` shows
the query commands **and** `backup`/`restore` together.

## Installation

```bash
aux4 aux4 pkger install aux4/db-mysql-backup
```

This pulls in `aux4/db-mysql` as a dependency, so the full `db:mysql` profile
(query commands + backup/restore) is available.

## Configuration

Connection details are supplied through a `config.yaml` profile so credentials
stay out of any backup catalog:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 3306
    database: shop
    user: root
    password: secret
```

Use it with `--configFile config.yaml --config prod`. Individual values can also
be passed directly with `--host`, `--port`, `--database`, `--user`,
`--password`, which take precedence over config values.

### Dump options

What goes into the dump is controlled by the same mechanism, so options can be
set once per profile and overridden per run on the command line:

```yaml
config:
  prod:
    host: 127.0.0.1
    database: shop
    user: root
    password: secret
    singleTransaction: true
    routines: true
    events: true
    maxAllowedPacket: 512M
    options: "--hex-blob"
```

| Option | Default | Effect |
|--------|---------|--------|
| `singleTransaction` | `true` | Dump in one transaction — a consistent InnoDB snapshot without locking tables |
| `routines` | `true` | Include stored procedures and functions |
| `events` | `true` | Include scheduled events |
| `triggers` | `true` | Include triggers |
| `noTablespaces` | `true` | Skip tablespace DDL, which needs the `PROCESS` privilege on MySQL 8.0+ |
| `defaultCharacterSet` | — | Character set for the dump |
| `maxAllowedPacket` | — | Maximum packet size, e.g. `512M` |
| `where` | — | Only dump rows matching a condition |
| `ignoreTable` | — | Exclude a table, as `db.table` |
| `options` | — | Additional raw `mysqldump` options, appended verbatim |

The defaults are chosen so a backup is complete and consistent out of the box.
`routines` and `events` matter in particular because **mysqldump omits stored
procedures, functions and scheduled events by default** — without them a restore
silently comes back missing those objects.

## backup

Creates a logical SQL dump of a MySQL database with `mysqldump`.

The destination path is resolved as:

- **`--path <file>`** — full path to write the dump (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither is given, the command fails fast. When the resolved path has no
extension, `.sql` is appended, so `aux4/backup` can pass an extension-less base
path and let this package name the artifact for its own format; an explicit
`--path dump.sql` is written exactly as given. If `mysqldump` fails, the
partial file is removed rather than left behind as a misleading artifact.

On success it prints a result manifest as a single line of JSON to stdout:

```bash
aux4 db mysql backup --configFile config.yaml --config prod --path /tmp/shop.sql
```

```text
{"path":"/tmp/shop.sql","bytes":1870,"checksum":"12b8fe1bd197b144ba5c9fa73dec60274efb56ce9ce8bc593307e71f763dd938","status":"success","format":"mysql-sqldump"}
```

The manifest (`path`, `bytes`, `checksum`, `status`, `format`) is consumed by
the `aux4/backup` orchestrator to catalog each run.

## restore

Loads a dump produced by `backup` back into a MySQL database by piping it into
the `mysql` client. The target database must already exist.

```bash
aux4 db mysql restore --configFile config.yaml --config prod --path /tmp/shop.sql
```

```text
{"path":"/tmp/shop.sql","status":"success"}
```

Path resolution (`--path` or `--dir` + `--file`) is identical to `backup`. If
the resolved file does not exist, the command fails fast.

## Binary resolution

Both commands locate the `mysqldump`/`mysql` binaries automatically:

1. Check `PATH` first (`command -v mysqldump` / `command -v mysql`).
2. Fall back to the Homebrew keg-only location
   (`$(brew --prefix mysql-client)/bin/...`).

This works on Linux (binaries on `PATH`) **and** on macOS (where `mysql-client`
is keg-only and not on `PATH`) with **no manual `PATH` export**. If a binary
cannot be found anywhere, the command fails with an install hint.

## System dependency

Requires the MySQL client tools (`mysqldump` and `mysql`):

- macOS: `brew install mysql-client`
- Debian/Ubuntu: `apt-get install default-mysql-client`
