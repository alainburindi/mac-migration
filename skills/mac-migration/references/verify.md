# Verify Migration — Phase 11

Run this on the **new Mac**. Read `~/mac-migration-state.md` first and compare every result with its audit summary and counts.

## 1. Tool versions

```bash
# MISSING means "not on PATH": check the state file to see whether it should be there
echo "=== Brew ===" && (command -v brew || echo "MISSING: brew not on PATH (check ~/.zprofile)")
echo "=== Node ===" && (node --version 2>/dev/null || echo MISSING)
echo "=== Python ===" && (python3 --version 2>/dev/null || echo MISSING)
echo "=== Ruby ===" && (ruby --version 2>/dev/null || echo MISSING)
echo "=== Go ===" && (go version 2>/dev/null || echo MISSING)
echo "=== Rust ===" && (rustc --version 2>/dev/null || echo MISSING)
echo "=== Java ===" && (java --version 2>/dev/null || echo MISSING)
echo "=== PHP ===" && (php --version 2>/dev/null || echo MISSING)
echo "=== Brew formulae ===" && brew leaves --installed-on-request 2>/dev/null | wc -l
echo "=== Brew casks ===" && brew list --cask 2>/dev/null | wc -l
```

## 2. Services running

```bash
brew services list
```
Every service that was running on the old Mac should show `started`.

## 3. Databases

```bash
# PostgreSQL (keg-only formulas: add $(brew --prefix PG_FORMULA)/bin to PATH first)
psql -d postgres -Atc "SELECT count(*) FROM pg_database WHERE NOT datistemplate AND datname <> 'postgres';"

# MongoDB
mongosh --quiet --eval "db.adminCommand({listDatabases:1}).databases.map(d => d.name)" 2>/dev/null

# MySQL
mysql -u root -N -e "SHOW DATABASES;" 2>/dev/null | grep -Ev '^(information_schema|performance_schema|sys|mysql)$'

# Redis
redis-cli ping 2>/dev/null && redis-cli dbsize
```
Compare with the counts recorded in Phase 5. If usernames differed, also check that no PostgreSQL objects are still owned by the old role:
```bash
psql -d postgres -Atc "SELECT count(*) FROM pg_database d JOIN pg_roles r ON r.oid = d.datdba WHERE r.rolname = 'OLD_USER';"
```

## 4. Projects

Use the new-username paths:
```bash
find "/Users/NEW_USER/code" "/Users/NEW_USER/work" -maxdepth 5 -name .git -type d -prune 2>/dev/null | wc -l
du -sh "/Users/NEW_USER/code" "/Users/NEW_USER/work"
```
The repo count should match. Sizes will be smaller by the excluded folders (`node_modules`, `.venv`, …) until Phase 10 rebuilds them.

## 5. SSH and GPG keys

```bash
ls ~/.ssh/
ssh -T -o BatchMode=yes -o ConnectTimeout=10 git@github.com 2>&1
gpg --list-secret-keys --keyid-format long 2>/dev/null
```

## 6. Shell startup time

```bash
for i in 1 2 3; do /usr/bin/time zsh -i -c exit; done
```
Should be under 0.5 seconds with the Phase 9 optimizations.

## 7. Claude Code

```bash
ls ~/.claude/projects/ | wc -l
```
The project count should match the old Mac.

## 8. Verification report

Produce a summary report, for example:

```
Migration Verification Report
==============================
✅ Node:        v24.7.0  (matches)
✅ Python:      3.11.11  (matches)
✅ Ruby:        3.4.1    (matches)
✅ Services:    5/5 running
✅ PostgreSQL:  29 databases
✅ MongoDB:     2 databases
✅ MySQL:       4 databases
✅ Projects:    43 repos
✅ SSH keys:    4 pairs
✅ Shell:       0.4s startup
✅ Claude Code: 8 projects

Migration complete!
```

Mark mismatches with ❌ and suggest fixes.

## 9. Clean up

Once the report is all ✅, offer the cleanup. Confirm each step with the user.

On the **new Mac** (Claude runs these):
```bash
# Remove the migration key from authorized_keys
sed -i '' '/ mac-migration$/d' ~/.ssh/authorized_keys
# Delete database dumps and transfer helpers (only after the user confirms data is verified)
rm -rf ~/db_backups ~/crontab.txt ~/vscode-extensions.txt
```
Remote Login can be turned off again in `System Settings > General > Sharing` if the user doesn't need it.

On the **old Mac** (the user runs these there, or in the old Mac's Claude session):
```bash
rm -f ~/.ssh/mac_migration_ed25519 ~/.ssh/mac_migration_ed25519.pub
sed -i '' '/^# >>> mac-migration/,/^# <<< mac-migration/d' ~/.ssh/config
rm -f ~/.ssh/config.pre-migration
```
Keep `~/db_backups` on the old Mac until the user is confident, or until the old Mac is wiped.

Mark Phase 11 done in the state file. It can stay as a record, or be deleted.
