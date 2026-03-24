# Database Migration Reference

## PostgreSQL

### Dump
```bash
# All databases + roles/users
pg_dumpall > ~/db_backups/pg_backup.sql

# Roles/users only (important for preserving permissions)
pg_dumpall --globals-only > ~/db_backups/pg_globals.sql

# Single database
pg_dump -U postgres DATABASE_NAME > ~/db_backups/DATABASE_NAME.sql
```

### Transfer
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/db_backups/pg_backup.sql ~/db_backups/pg_globals.sql \
  NEW_USER@NEW_MAC_IP:~/db_backups/
```

### Restore
```bash
# Roles first, then databases
psql -U NEW_USER -d postgres < ~/db_backups/pg_globals.sql
psql -U NEW_USER -d postgres < ~/db_backups/pg_backup.sql
```

### Notes
- Database owners will reference the old username — restore globals first to create the roles
- If the username changed: `REASSIGN OWNED BY old_user TO new_user;` per database

---

## MySQL / MariaDB

### Dump
```bash
# All databases (redirect stderr to avoid warning messages in dump file)
mysqldump -u root -p \
  --column-statistics=0 \
  --no-tablespaces \
  --all-databases \
  2>/dev/null > ~/db_backups/mysql_backup.sql

# Single database
mysqldump -u root -p --column-statistics=0 DATABASE_NAME 2>/dev/null > ~/db_backups/DATABASE_NAME.sql
```

### Transfer
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/db_backups/mysql_backup.sql \
  NEW_USER@NEW_MAC_IP:~/db_backups/
```

### Restore
```bash
mysql -u root -p < ~/db_backups/mysql_backup.sql
```

### MySQL 5.7 → 8.0 Compatibility Issues
- `--column-statistics=0` flag required (5.7 mysqldump doesn't support the column stats query used by 8.0 client)
- `@@global.character_set_client` may be NULL in old dumps — fix with: `sed 's/@@global.character_set_client/utf8mb4/g'`
- Some stored procedures may need manual review

---

## MongoDB

### Dump
```bash
mongodump --out ~/db_backups/mongo_backup/

# Single database
mongodump --db DATABASE_NAME --out ~/db_backups/mongo_backup/
```

### Transfer
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/db_backups/mongo_backup/ \
  NEW_USER@NEW_MAC_IP:~/db_backups/mongo_backup/
```

### Restore
```bash
mongorestore ~/db_backups/mongo_backup/

# Single database
mongorestore --db DATABASE_NAME ~/db_backups/mongo_backup/DATABASE_NAME/
```

---

## Redis

### Dump
```bash
# Option 1: copy the RDB file
cp /opt/homebrew/var/db/redis/dump.rdb ~/db_backups/redis_dump.rdb

# Option 2: trigger a save and copy
redis-cli SAVE && cp /opt/homebrew/var/db/redis/dump.rdb ~/db_backups/redis_dump.rdb

# Option 3: export as RDB via CLI
redis-cli --rdb ~/db_backups/redis_dump.rdb
```

### Transfer
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/db_backups/redis_dump.rdb \
  NEW_USER@NEW_MAC_IP:~/db_backups/
```

### Restore
```bash
# Stop Redis, replace dump.rdb, restart
brew services stop redis
cp ~/db_backups/redis_dump.rdb /opt/homebrew/var/db/redis/dump.rdb
brew services start redis
```

---

## SQLite

SQLite databases are just files — copy them directly:

```bash
# Find SQLite files in projects
find $PROJECT_DIRS -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null

# They are copied automatically as part of the project rsync in Phase 4
```

---

## CockroachDB

```bash
# Dump
cockroach dump DATABASE_NAME --insecure > ~/db_backups/cockroach_backup.sql

# Restore
cockroach sql --insecure < ~/db_backups/cockroach_backup.sql
```

---

## General Tips

- Always check database sizes before dumping: large databases may need external storage
- Run dumps while databases are idle if possible to avoid inconsistent state
- Test restore on a small database first before restoring everything
- Keep backup files until you've verified the new Mac works correctly
