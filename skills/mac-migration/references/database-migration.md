# Database Migration — Phase 5

**Before starting:**
- The setup script (Phase 4) must have finished on the new Mac, so the database engines are installed and running there.
- Dump commands run on the **old Mac**. Restore commands run on the new Mac through `ssh newmac`, with Homebrew loaded first (see Ground rules in SKILL.md).
- The new Mac's major version must be the same or newer than the old Mac's for every engine.
- Dumps and transfers of large databases (more than a few hundred MB) can outlast the Bash tool's timeout. Run them in the background with output going to log files (see Long-running commands in SKILL.md).

Steps for each engine:
1. List the databases with their sizes.
2. The user picks which ones to migrate.
3. Dump them, verify the dump, transfer, restore.
4. Record database counts in the state file.

```bash
mkdir -p ~/db_backups
```

---

## PostgreSQL

Homebrew's PostgreSQL has no `postgres` role. The superuser is the macOS username, so commands run without `-U`. Versioned formulas (`PG_FORMULA`, e.g. `postgresql@16`, from the state file) are keg-only: on the new Mac, put `$(brew --prefix PG_FORMULA)/bin` on PATH before using `psql`.

### List
```bash
psql -d postgres -Atc "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE NOT datistemplate AND datname <> 'postgres' ORDER BY pg_database_size(datname) DESC;"
```

### Dump
Dump roles once, then each selected database in custom format:
```bash
mkdir -p ~/db_backups/pg
pg_dumpall --globals-only > ~/db_backups/pg/globals.sql 2> ~/db_backups/pg/dump.log
for db in "app_dev" "app_test"; do
  pg_dump -Fc -f ~/db_backups/pg/"$db".dump "$db" 2>> ~/db_backups/pg/dump.log
done
cat ~/db_backups/pg/dump.log   # should be empty
ls -lh ~/db_backups/pg/
```

### Transfer
```bash
rsync -a --partial ~/db_backups/pg/ newmac:db_backups/pg/
```

### Restore
Restore roles first, then each database. "role ... already exists" errors from `globals.sql` are harmless.
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
export PATH="$(brew --prefix PG_FORMULA)/bin:$PATH"
psql -d postgres -f ~/db_backups/pg/globals.sql
for f in ~/db_backups/pg/*.dump; do
  pg_restore --create -d postgres "$f" || echo "FAILED: $f"
done
EOF
```

### If usernames differ
Objects are still owned by the old user's role, which `globals.sql` recreated. Hand everything to the new user in **every** restored database:
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
export PATH="$(brew --prefix PG_FORMULA)/bin:$PATH"
for db in "app_dev" "app_test"; do
  psql -d "$db" -c 'REASSIGN OWNED BY "OLD_USER" TO "NEW_USER";'
done
EOF
```
Only once every database is reassigned, the old role can be dropped (optional): `DROP OWNED BY "OLD_USER";` in each database, then `DROP ROLE "OLD_USER";`.

---

## MySQL / MariaDB

Check the client version first. `--column-statistics=0` exists only in MySQL 8.x `mysqldump`, where it's needed to dump a 5.7 server. MySQL 5.7 and MariaDB clients fail with `unknown variable 'column-statistics=0'`.
```bash
mysqldump --version
```

Homebrew MySQL's `root` has no password by default. If one is set, add `-p` and have the user run the command themselves with the `!` prefix.

### List
```bash
mysql -u root -e "SELECT table_schema AS db, ROUND(SUM(data_length + index_length)/1024/1024, 1) AS mb FROM information_schema.tables WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys') GROUP BY table_schema ORDER BY mb DESC;"
```

### Dump
Dump user databases only. Never include the `mysql` system schema: restoring 5.7's system tables into 8.0 breaks the server.

Users and grants are not included. List them with `SELECT user, host FROM mysql.user;` and `SHOW GRANTS FOR 'user'@'host';`, then recreate them on the new Mac.
```bash
# All user databases (add --column-statistics=0 only if mysqldump is 8.x)
mysqldump -u root \
  --no-tablespaces --routines --events \
  --databases $(mysql -u root -N -e "SHOW DATABASES" | grep -Ev '^(mysql|information_schema|performance_schema|sys)$') \
  > ~/db_backups/mysql_backup.sql 2> ~/db_backups/mysql_dump.log

# Or selected databases only
mysqldump -u root --no-tablespaces --routines --events --databases "app_dev" "app_test" \
  > ~/db_backups/mysql_backup.sql 2> ~/db_backups/mysql_dump.log
```

Verify the dump before moving on. Errors go to the log, not the terminal:
```bash
cat ~/db_backups/mysql_dump.log                # should be empty or warnings only
tail -1 ~/db_backups/mysql_backup.sql          # must be "-- Dump completed on ..."
```

### Transfer
```bash
rsync -a --partial ~/db_backups/mysql_backup.sql newmac:db_backups/
```

### Restore
From 5.7: first apply the compatibility fixes below to the dump. The `mysql` client stops at the first error, so log stderr and check it. An error means a partial restore.
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
mysql -u root < ~/db_backups/mysql_backup.sql 2> ~/db_backups/mysql_restore.log
echo "exit: $?"
cat ~/db_backups/mysql_restore.log
EOF
```
Versioned formulas (`mysql@8.4`) are keg-only: add `export PATH="$(brew --prefix mysql@8.4)/bin:$PATH"` after the `eval` line.

### MySQL 5.7 → 8.0 compatibility
- `--column-statistics=0` is only for an 8.x `mysqldump` reading a 5.7 server. Never pass it to a 5.7 or MariaDB `mysqldump`.
- 5.7 dumps of routines and triggers set `sql_mode` with `NO_AUTO_CREATE_USER`, which 8.0 rejects (ERROR 1231). Remove it before restoring: `sed -i '' -E 's/,?NO_AUTO_CREATE_USER//g' ~/db_backups/mysql_backup.sql`
- `@@global.character_set_client` may be NULL in old dumps. Fix with: `sed -i '' 's/@@global.character_set_client/utf8mb4/g' ~/db_backups/mysql_backup.sql`
- Plain `brew install mysql` is now 9.x, which has dropped `mysql_native_password`. Use `mysql@8.4`. Recreated app users must use `caching_sha2_password`, and old client libraries may need upgrading.
- Some stored procedures may need manual review.

---

## MongoDB

### List
```bash
mongosh --quiet --eval 'db.adminCommand({listDatabases:1}).databases.forEach(d => print(d.name, (d.sizeOnDisk/1e6).toFixed(1) + " MB"))'
```

### Dump
```bash
# All databases
mongodump --out ~/db_backups/mongo/ 2> ~/db_backups/mongo_dump.log

# Single database
mongodump --db DATABASE_NAME --out ~/db_backups/mongo/ 2> ~/db_backups/mongo_dump.log
```
Check `~/db_backups/mongo_dump.log` for lines containing `Failed` or `error`.

### Transfer
```bash
rsync -a --partial ~/db_backups/mongo/ newmac:db_backups/mongo/
```

### Restore
Skip the `admin`, `config` and `local` system databases:
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
mongorestore --nsExclude='admin.*' --nsExclude='config.*' --nsExclude='local.*' ~/db_backups/mongo/
# Single database: mongorestore --nsInclude='DATABASE_NAME.*' ~/db_backups/mongo/
EOF
```

---

## Redis

### Dump
`--rdb` writes a fresh snapshot wherever Redis keeps its data, so no path guessing is needed:
```bash
redis-cli --rdb ~/db_backups/redis_dump.rdb
```

### Transfer
```bash
rsync -a ~/db_backups/redis_dump.rdb newmac:db_backups/
```

### Restore
Find the data directory, stop Redis, swap in the file, then start Redis again. If `appendonly` is `yes` on the new Mac, Redis loads the AOF instead of the RDB. Temporarily set `appendonly no` in `redis.conf` for the first start, then turn it back on with `redis-cli CONFIG SET appendonly yes`.
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
dir=$(redis-cli CONFIG GET dir | tail -1)
redis-cli CONFIG GET appendonly
brew services stop redis
cp ~/db_backups/redis_dump.rdb "$dir/dump.rdb"
brew services start redis
EOF
```

---

## SQLite

SQLite databases are plain files. They're copied with the projects in Phase 4 (Mode A), which also told the user to stop the apps writing to them. List them across **all** project directories, so the user can confirm none live outside them (and so Mode B users can copy them separately):
```bash
find "/Users/OLD_USER/code" "/Users/OLD_USER/work" \( -name node_modules -o -name .git \) -prune -o -type f \( -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" \) -print 2>/dev/null
```

---

## CockroachDB

`cockroach dump` was removed in v20.2. Use `BACKUP`/`RESTORE` with `nodelocal`, which writes to `<store-dir>/extern/`:
```bash
# Old Mac
cockroach sql --insecure -e "BACKUP DATABASE DATABASE_NAME INTO 'nodelocal://1/migration';"
# Copy <store-dir>/extern/migration/ to the same place on the new Mac, then on the new Mac:
cockroach sql --insecure -e "RESTORE DATABASE DATABASE_NAME FROM LATEST IN 'nodelocal://1/migration';"
```

---

## Databases in Docker

List the volumes and ask which ones hold data worth keeping:
```bash
docker volume ls
docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Mounts}}'
```

- **Database containers (Postgres, MySQL, Mongo):** dump through the container with the engine's own tool. Raw data files are not guaranteed portable across architectures and major versions.
  ```bash
  mkdir -p ~/db_backups/docker
  docker exec CONTAINER pg_dumpall -U postgres > ~/db_backups/docker/CONTAINER.sql
  ```
- **Other volumes (uploads, caches, app state):** stop the containers using the volume, then archive it:
  ```bash
  mkdir -p ~/db_backups/docker
  docker run --rm -v VOLUME:/from -v ~/db_backups/docker:/to alpine tar czf /to/VOLUME.tgz -C /from .
  ```

Transfer everything:
```bash
rsync -a --partial ~/db_backups/docker/ newmac:db_backups/docker/
```

Restore on the new Mac. Docker Desktop must be running there, and the project's containers must be started first (e.g. `docker compose up -d` in the project folder) so the containers and volumes exist. Docker Desktop's CLI lives in `/usr/local/bin`, which a non-interactive SSH session may not have on PATH:
```bash
ssh newmac 'bash -s' <<'EOF'
export PATH="/usr/local/bin:/Applications/Docker.app/Contents/Resources/bin:$PATH"
# Database container
docker exec -i CONTAINER psql -U postgres < ~/db_backups/docker/CONTAINER.sql
# Plain volume (stop containers using it first)
docker volume create VOLUME
docker run --rm -v VOLUME:/to -v ~/db_backups/docker:/from alpine tar xzf /from/VOLUME.tgz -C /to
EOF
```

---

## Database configuration

Ask: "Do you want to copy database configuration settings (e.g. `max_locks_per_transaction`, `max_connections`) from your old Mac? Recommended if you've tuned your databases."

**PostgreSQL:** show settings that came from the config file:
```bash
psql -d postgres -c "SELECT name, setting, unit FROM pg_settings WHERE source = 'configuration file' ORDER BY name;"
```
For each non-default value the user wants to keep, set it on the new Mac and restart. `ALTER SYSTEM` writes to `postgresql.auto.conf`, so no file paths are needed:
```bash
ssh newmac 'bash -s' <<'EOF'
eval "$(/opt/homebrew/bin/brew shellenv)"
export PATH="$(brew --prefix PG_FORMULA)/bin:$PATH"
psql -d postgres -c "ALTER SYSTEM SET max_locks_per_transaction = 256;"
brew services restart PG_FORMULA
EOF
```

**MySQL:**
```bash
/usr/bin/grep -viE 'passw|secret|token' "$(brew --prefix)/etc/my.cnf" 2>/dev/null
```
Copy the custom settings into `/opt/homebrew/etc/my.cnf` on the new Mac, then run `brew services restart mysql` there.

**MongoDB:**
```bash
cat "$(brew --prefix)/etc/mongod.conf" 2>/dev/null
```
Copy the custom settings (not `dbPath`/`path`, which differ per Mac) into `/opt/homebrew/etc/mongod.conf` on the new Mac, then run `brew services restart mongodb-community` there.

---

## General tips

- Check database sizes before dumping. Large databases may need external storage, or a direct pipe: `pg_dump -Fc DB | ssh newmac 'cat > db_backups/DB.dump'`.
- Stop the apps that write to a database before dumping it.
- Test a restore on a small database first.
- Keep the dumps on both Macs until Phase 11 has verified the new Mac.
