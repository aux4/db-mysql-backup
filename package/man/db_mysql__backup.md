#### Description

The `backup` command creates a logical SQL dump of a MySQL database by invoking the `mysqldump` system CLI. It is a backup **provider**: connection details come from a `config.yaml` profile (via `--config`/`--configFile`) so credentials stay out of any catalog, and the destination is resolved from a path.

This command is contributed by the `aux4/db-mysql-backup` package, which extends the `db:mysql` profile owned by `aux4/db-mysql`. It is packaged separately so the `mysqldump`/`mysql` system dependency is only required when backup/restore is actually used — the core `aux4/db-mysql` query commands need no external CLIs.

The destination path is resolved in this order:

- **`--path <file>`** — the full path to write the dump to (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither `--path` nor a `--dir`/`--file` pair is provided, the command fails fast with a clear error and a non-zero exit code. When the resolved path has no extension, `.sql` is appended — this lets `aux4/backup` pass an extension-less base path and leave artifact naming to the provider, while an explicit `--path dump.sql` is still written exactly as given.

**Dump options.** What goes into the dump is controlled by variables, so they can be set once in the `config.yaml` profile (and overridden per run on the command line):

| Option | Default | mysqldump flag |
|--------|---------|----------------|
| `--singleTransaction` | `true` | `--single-transaction` |
| `--routines` | `true` | `--routines` |
| `--events` | `true` | `--events` |
| `--triggers` | `true` | `--skip-triggers` when `false` |
| `--noTablespaces` | `true` | `--no-tablespaces` |
| `--defaultCharacterSet` | — | `--default-character-set` |
| `--maxAllowedPacket` | — | `--max-allowed-packet` |
| `--where` | — | `--where` |
| `--ignoreTable` | — | `--ignore-table` |
| `--options` | — | appended verbatim |

The defaults are chosen so a backup is complete and consistent out of the box. `--singleTransaction` dumps inside one transaction, giving a consistent InnoDB snapshot without locking tables. `--routines` and `--events` matter because **mysqldump omits stored procedures, functions and scheduled events by default** — without them a restore silently comes back missing those objects. `--noTablespaces` avoids the `PROCESS` privilege requirement on MySQL 8.0+.

If `mysqldump` fails, the partially written file is removed rather than left behind as a misleading artifact.

After writing the dump, `backup` prints a **result manifest** as a single line of JSON to stdout:

```json
{
  "path": "<resolved path>",
  "bytes": "<file size>",
  "checksum": "<sha256>",
  "status": "success",
  "format": "mysql-sqldump"
}
```

The `bytes` value is computed with `wc -c` and the `checksum` with `shasum -a 256` (falling back to `sha256sum`). The manifest is consumed by the `aux4/backup` orchestrator to catalog each run.

**Binary resolution:** the command locates `mysqldump` automatically. It first checks `PATH` (`command -v mysqldump`), and if not found falls back to the Homebrew keg-only location (`$(brew --prefix mysql-client)/bin/mysqldump`). This means it works on Linux (where `mysqldump` is on `PATH`) and on macOS (where `mysql-client` is keg-only and not on `PATH`) **without** any manual `PATH` export. If the binary cannot be found anywhere, the command fails with an install hint.

**System dependency:** requires the `mysqldump` client. Install with `brew install mysql-client` (macOS) or `apt-get install default-mysql-client` (Debian/Ubuntu).

#### Usage

```bash
aux4 db mysql backup --configFile config.yaml --config <profile> --path <file>
aux4 db mysql backup --configFile config.yaml --config <profile> --dir <directory> --file <name>
```

--host      Database host (default: localhost)
--port      Database port (default: 3306)
--database  Database name (default: mysql)
--user      Database user (default: root)
--password  Database password
--path      Full path to write the dump file (takes precedence over --dir/--file)
--dir       Directory to write the dump file (combined with --file)
--file      File name for the dump (combined with --dir)

Dump options: --singleTransaction, --routines, --events, --triggers, --noTablespaces
              --defaultCharacterSet, --maxAllowedPacket, --where, --ignoreTable, --options

Configuration file — dump options live alongside the connection settings:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 3306
    database: shop
    user: root
    password: secret
    singleTransaction: true
    routines: true
    events: true
    maxAllowedPacket: 512M
```

#### Example

```bash
aux4 db mysql backup --configFile config.yaml --config prod --path /tmp/shop.sql
```

```text
{"path":"/tmp/shop.sql","bytes":1870,"checksum":"12b8fe1bd197b144ba5c9fa73dec60274efb56ce9ce8bc593307e71f763dd938","status":"success","format":"mysql-sqldump"}
```
