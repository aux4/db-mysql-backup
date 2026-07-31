#### Description

The `restore` command loads a SQL dump produced by `aux4 db mysql backup` back into a MySQL database by piping the file into the `mysql` client. It is the counterpart of the `backup` provider command and uses the same connection and path resolution.

This command is contributed by the `aux4/db-mysql-backup` package, which extends the `db:mysql` profile owned by `aux4/db-mysql`. It is packaged separately so the `mysqldump`/`mysql` system dependency is only required when backup/restore is actually used.

Connection details come from a `config.yaml` profile (via `--config`/`--configFile`). The dump path is resolved in this order:

- **`--path <file>`** — the full path to the dump to restore (takes precedence).
- **`--dir <directory>` + `--file <name>`** — combined as `<dir>/<file>`.

If neither `--path` nor a `--dir`/`--file` pair is provided, or the resolved file does not exist, the command fails fast with a clear error and a non-zero exit code.

The target database (from the config profile) must already exist — the dump only contains table-level statements. On success `restore` prints a small outcome JSON to stdout:

```json
{
  "path": "<resolved path>",
  "status": "success"
}
```

**Binary resolution:** the command locates the `mysql` client automatically. It first checks `PATH` (`command -v mysql`), and if not found falls back to the Homebrew keg-only location (`$(brew --prefix mysql-client)/bin/mysql`). This means it works on Linux (where `mysql` is on `PATH`) and on macOS (where `mysql-client` is keg-only and not on `PATH`) **without** any manual `PATH` export. If the binary cannot be found anywhere, the command fails with an install hint.

**System dependency:** requires the `mysql` client. Install with `brew install mysql-client` (macOS) or `apt-get install default-mysql-client` (Debian/Ubuntu).

#### Usage

```bash
aux4 db mysql restore --configFile config.yaml --config <profile> --path <file>
aux4 db mysql restore --configFile config.yaml --config <profile> --dir <directory> --file <name>
```

--host      Database host (default: localhost)
--port      Database port (default: 3306)
--database  Database name (default: mysql)
--user      Database user (default: root)
--password  Database password
--path      Full path to the dump file to restore (takes precedence over --dir/--file)
--dir       Directory containing the dump file (combined with --file)
--file      File name of the dump (combined with --dir)

Configuration file:

```yaml
config:
  prod:
    host: 127.0.0.1
    port: 3306
    database: shop
    user: root
    password: secret
```

#### Example

```bash
aux4 db mysql restore --configFile config.yaml --config prod --path /tmp/shop.sql
```

```text
{"path":"/tmp/shop.sql","status":"success"}
```
