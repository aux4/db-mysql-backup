# db mysql backup and restore

Provider commands (contributed by `aux4/db-mysql-backup`) that wrap
`mysqldump`/`mysql`. Connection is supplied via `--config` (a `config.yaml`
profile) and the destination via `--path` (or `--dir` + `--file`). `backup`
prints a result manifest to stdout; `restore` loads the dump back into the
database. The `mysqldump`/`mysql` binaries are resolved automatically (PATH or
the Homebrew keg-only location) — no manual `PATH` export is required.

```file:config.yaml
config:
  test:
    host: 127.0.0.1
    port: 3306
    database: bkptest
    user: root
    password: mysecretpassword
  noroutines:
    host: 127.0.0.1
    port: 3306
    database: bkptest
    user: root
    password: mysecretpassword
    routines: false
```

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest"
```

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "CREATE DATABASE bkptest"
```

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --database bkptest --user root --password mysecretpassword --query "CREATE TABLE items (id INT PRIMARY KEY, name TEXT, qty INTEGER)"
```

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --database bkptest --user root --password mysecretpassword --query "INSERT INTO items VALUES (1, 'apple', 10), (2, 'pear', 20), (3, 'plum', 30)"
```


```afterAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest"
```

```afterAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest_restore"
```

```afterAll
rm -f /tmp/aux4-bkp-test.sql /tmp/aux4-bkp-dirfile.sql /tmp/aux4-bkp-noroutines.sql /tmp/aux4-bkp-clioverride.sql /tmp/aux4-bkp-opts.sql /tmp/aux4-bkp-failed.sql
```

## backup with --config and --path

### should write the dump and print a manifest

```execute
aux4 db mysql backup --configFile config.yaml --config test --path /tmp/aux4-bkp-test.sql
```

```expect:regex
\{"bytes":"\d+","checksum":"[a-f0-9]{64}","format":"mysql-sqldump","path":"/tmp/aux4-bkp-test\.sql","status":"success"\}
```

### should create a non-empty dump file

```execute
test -s /tmp/aux4-bkp-test.sql && echo present
```

```expect
present
```

## backup with --dir and --file

### should resolve the path from dir + file

```execute
aux4 db mysql backup --configFile config.yaml --config test --dir /tmp --file aux4-bkp-dirfile.sql
```

```expect:regex
\{"bytes":"\d+","checksum":"[a-f0-9]{64}","format":"mysql-sqldump","path":"/tmp/aux4-bkp-dirfile\.sql","status":"success"\}
```

## dump options

### should dump routines and events by default

```execute
grep -c "Dumping routines\|Dumping events" /tmp/aux4-bkp-test.sql
```

```expect
2
```

### should omit routines when the config profile disables them

```execute
aux4 db mysql backup --configFile config.yaml --config noroutines --path /tmp/aux4-bkp-noroutines.sql >/dev/null
grep -c "Dumping routines" /tmp/aux4-bkp-noroutines.sql || true
```

```expect
0
```

### should let a CLI flag override the config profile

```execute
aux4 db mysql backup --configFile config.yaml --config test --routines false --path /tmp/aux4-bkp-clioverride.sql >/dev/null
grep -c "Dumping routines" /tmp/aux4-bkp-clioverride.sql || true
```

```expect
0
```

### should accept value-bearing options from the config profile

```execute
aux4 db mysql backup --configFile config.yaml --config test --maxAllowedPacket 512M --defaultCharacterSet utf8mb4 --path /tmp/aux4-bkp-opts.sql
```

```expect:regex
\{"bytes":"\d+","checksum":"[a-f0-9]{64}","format":"mysql-sqldump","path":"/tmp/aux4-bkp-opts\.sql","status":"success"\}
```

## backup failure

### should not leave a partial dump file behind

```execute
aux4 db mysql backup --configFile config.yaml --config test --password WRONGPASSWORD --path /tmp/aux4-bkp-failed.sql 2>/dev/null
test -e /tmp/aux4-bkp-failed.sql && echo "leftover" || echo "cleaned up"
```

```expect
cleaned up
```

## backup with no path

### should fail fast when neither path nor dir/file is given

```execute
aux4 db mysql backup --configFile config.yaml --config test
```

```error:partial
Error: provide --path, or --dir and --file
```

## restore into the same database

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --database bkptest --user root --password mysecretpassword --query "DROP TABLE items"
```

### should restore the dump and print an outcome

```execute
aux4 db mysql restore --configFile config.yaml --config test --path /tmp/aux4-bkp-test.sql
```

```expect
{"path":"/tmp/aux4-bkp-test.sql","status":"success"}
```

### should bring the rows back

```execute
aux4 db mysql execute --host 127.0.0.1 --port 3306 --database bkptest --user root --password mysecretpassword --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore into a fresh database

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "DROP DATABASE IF EXISTS bkptest_restore"
```

```beforeAll
aux4 db mysql execute --host 127.0.0.1 --port 3306 --user root --password mysecretpassword --query "CREATE DATABASE bkptest_restore"
```

### should restore the dump into a different database

```execute
aux4 db mysql restore --configFile config.yaml --config test --database bkptest_restore --path /tmp/aux4-bkp-test.sql
```

```expect
{"path":"/tmp/aux4-bkp-test.sql","status":"success"}
```

### should have the rows in the fresh database

```execute
aux4 db mysql execute --host 127.0.0.1 --port 3306 --database bkptest_restore --user root --password mysecretpassword --query "SELECT * FROM items ORDER BY id" | jq -c .
```

```expect
[{"id":1,"name":"apple","qty":10},{"id":2,"name":"pear","qty":20},{"id":3,"name":"plum","qty":30}]
```

## restore with a missing file

### should fail fast when the dump file does not exist

```execute
aux4 db mysql restore --configFile config.yaml --config test --path /tmp/does-not-exist.sql
```

```error:partial
Error: backup file not found: /tmp/does-not-exist.sql
```
