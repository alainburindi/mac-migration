---
name: mac-migration
description: Guide the user through migrating their complete developer environment from an old Mac to a new Mac. Use this when the user says "migrate my Mac", "setting up a new Mac", "transferring to new Mac", "Mac migration", or mentions moving dev tools between Macs.
argument-hint: "[audit|git-check|generate-script|transfer|databases|apps|dotfiles|claude-data|shell|auth|verify]"
allowed-tools: [Bash, Read, Write, Glob, Grep]
---

# Mac Migration

Help the user migrate their complete developer setup from an old Mac to a new Mac **without using Migration Assistant**.

This skill is specifically designed for situations where Migration Assistant is not an option:
- **Different usernames** between old and new Mac (e.g. personal vs work machine)
- **Work/corporate machines** with assigned usernames or MDM restrictions
- **Wanting full control** over exactly what gets migrated

This is a 10-phase interactive process. Work through each phase conversationally:
run checks, show what was found, confirm with the user, then generate outputs.

**Always show commands before running them. Never run destructive commands without confirmation.**

## Arguments

If invoked with `$ARGUMENTS`, jump to the corresponding phase:
- `audit` → Phase 1
- `git-check` → Phase 2
- `generate-script` → Phase 3
- `transfer` → Phase 4
- `databases` → Phase 5
- `apps` → Phase 6
- `dotfiles` → Phase 7
- `claude-data` → Phase 8
- `shell` → Phase 9
- `auth` → Phase 10
- `verify` → Phase 11
- (empty) → Start from welcome message

## Welcome Message

When no arguments are given, greet the user and show the checklist:

```
Mac Migration — 10 Phases

[ ] Phase 1:  Audit old Mac
[ ] Phase 2:  Check unpushed git work
[ ] Phase 3:  Generate setup script
[ ] Phase 4:  Transfer files
[ ] Phase 5:  Migrate databases
[ ] Phase 6:  Copy apps and data
[ ] Phase 7:  Copy dotfiles
[ ] Phase 8:  Migrate Claude Code data
[ ] Phase 9:  Optimize shell (if needed)
[ ] Phase 10: Re-authenticate
[ ] Phase 11: Verify migration
```

Then ask:
1. "Are you starting from the beginning or resuming? If resuming, which phase?"
2. "What is your username on the **old** Mac?" (e.g. `john`)
3. "What is your username on the **new** Mac?" (e.g. `johnsmith`) — note if they differ, this will affect path fixing in later phases
4. "Do you already have SSH access set up between the two Macs?"

Store these answers and reference them throughout all phases.

---

## Phase 1: Audit Old Mac

Read `references/audit-commands.md` for the complete command set.

### Steps:
1. Ask the user to run the audit commands on the old Mac and paste the output, OR offer to run them via SSH if access is already set up.
2. Parse the output and produce a structured summary table:
   - Languages & runtimes detected (with versions)
   - Package managers and their global packages
   - Databases running
   - Cloud CLIs installed
   - Shell configuration
   - SSH keys found
3. Ask: "Does anything look missing or wrong?"
4. Save the summary — it will be used to generate the setup script in Phase 3.

### Key things to detect:
- **Any language runtime** — Node (nvm/fnm/brew), Python (pyenv/brew), Ruby (rbenv/rvm/brew), Go, Rust, Java (jenv/brew), PHP, Elixir, Kotlin, Swift, Dart/Flutter, R, etc.
- **Homebrew** — `brew leaves` for user-installed formulae, `brew list --cask` for GUI apps
- **Package managers** — npm globals, pip globals, gem globals, cargo installs, go installs
- **Databases** — detect any running: PostgreSQL, MySQL, MongoDB, Redis, SQLite, MariaDB, CockroachDB, Cassandra, etc.
- **Cloud & DevOps CLIs** — AWS, GCP, Azure, Fly.io, Heroku, Vercel, Netlify, Stripe, Terraform, kubectl, helm, etc.
- **Mobile dev** — Xcode, Android Studio, cocoapods, flutter, fastlane
- **Shell** — zsh, bash, fish — and what's loading in the config

---

## Phase 2: Check Unpushed Git Work

### Steps:
1. Ask: "Where do you store your projects? (e.g. `~/Documents/work`, `~/Projects`, `~/code` — you can list multiple directories separated by spaces)"
2. Store the answer as PROJECT_DIRS — use it in all subsequent phases that reference project locations.
3. Scan each directory for git repos with issues:

```bash
find $PROJECT_DIRS -maxdepth 4 -name ".git" -type d 2>/dev/null | sed 's|/.git||' | while read repo; do
  name=$(basename "$repo")
  unpushed=$(git -C "$repo" log --oneline --branches --not --remotes 2>/dev/null | wc -l | tr -d ' ')
  dirty=$(git -C "$repo" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
  local_only=$(git -C "$repo" branch -vv 2>/dev/null | grep -v '\[' | wc -l | tr -d ' ')
  if [ "$unpushed" -gt 0 ] || [ "$dirty" -gt 0 ] || [ "$local_only" -gt 0 ]; then
    echo "⚠️  $name"
    [ "$unpushed" -gt 0 ] && echo "   unpushed commits: $unpushed"
    [ "$dirty" -gt 0 ] && echo "   uncommitted changes: $dirty files"
    [ "$local_only" -gt 0 ] && echo "   local-only branches: $local_only"
  fi
done
```

3. Show the results and ask the user to push or acknowledge each repo before continuing.
4. Suggest: for repos that can't be pushed (uncommitted WIP), use rsync to copy them as-is — git history and local branches are preserved.

---

## Phase 3: Generate Setup Script

Read `references/setup-script-template.md` for the scaffold.

### Steps:
1. Based on Phase 1 audit output, generate a personalized `new-mac-setup.sh`
2. Include only tools actually detected on the old Mac
3. Present the script to the user and ask: "Does this look right? Any tools to add or remove?"
4. Save the script to `~/new-mac-setup.sh` on the old Mac so it can be transferred

### Key generation rules:
- Use `brew install` for CLI tools, `brew install --cask` for GUI apps
- For PHP versions older than what brew supports natively, use `shivammathur/php` tap
- For MySQL 5.7 (EOL), upgrade to MySQL 8 and warn the user
- Include `nvm install X` for each Node version detected
- Include `pyenv install X` for each Python version detected
- Do the same for rbenv, jenv, rustup, etc.
- Add global package reinstalls (npm -g, pip, gem, cargo)
- Start all detected database services at the end
- Write a clean `.zshrc` with correct paths for the new username

---

## Phase 4: Transfer Files

### Steps:

**1. Connection method**

Ask: "How do you want to connect the two Macs?"
- **Option A — WiFi** (same network, rsync over SSH): Ask for old Mac's IP (`System Settings > Wi-Fi > Details`) and enable Remote Login (`System Settings > General > Sharing > Remote Login`)
- **Option B — Thunderbolt/USB-C cable** (fastest): Connect directly, check `System Settings > Network` for Thunderbolt Bridge IP. Warn that the cable must support Thunderbolt (look for ⚡ symbol) — charge-only cables won't work.

**2. Set up SSH key auth** (if not already done):
```bash
ssh-copy-id -o IdentitiesOnly=yes -i ~/.ssh/id_rsa.pub NEW_USER@NEW_MAC_IP
```

**3. Transfer directories**

Ask which directories to transfer. Common ones:
- All directories from PROJECT_DIRS (set in Phase 2)
- `~/Desktop`
- `~/Downloads`
- `~/Pictures`
- `~/Movies`
- `~/Music`
- Any project directories from Phase 2

Generate rsync commands for each:
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  PROJECT_DIR/ NEW_USER@NEW_MAC_IP:PROJECT_DIR/
```

**4. Username mismatch warning**

If old and new usernames differ, warn: "Your dotfiles and configs contain hardcoded paths with your old username. We'll fix these in Phase 7 and Phase 8."

**5. Resumability**

Remind: "rsync is safe to re-run — it skips already transferred files. If interrupted, just run the same command again."

---

## Phase 5: Migrate Databases

Read `references/database-migration.md` for engine-specific commands.

### Steps:
1. Based on Phase 1 audit, list all detected database engines
2. Ask: "Which databases do you want to migrate? Should I list them first?"
3. For each engine, list the databases with their sizes before dumping
4. Let the user select which databases to include
5. Generate dump commands, transfer via rsync, then restore commands

### Engine detection and handling:
- **PostgreSQL** — `pg_dumpall` for all, `pg_dump` for individual DBs. Export roles separately with `pg_dumpall --globals-only`. Note owner username will change if usernames differ.
- **MySQL/MariaDB** — Use `--column-statistics=0 --no-tablespaces` flags. Warn about 5.7→8.0 charset compatibility. Redirect stderr to avoid warning messages in the dump file.
- **MongoDB** — `mongodump --out ~/mongo_backup/` then `mongorestore`
- **Redis** — Copy `dump.rdb` file or use `redis-cli --rdb`
- **SQLite** — Just copy the `.db` files directly
- **Any other engine** — Ask the user for the dump/restore commands specific to their version

---

## Phase 6: Copy Apps and App Data

### Steps:

**1. List installed apps**
```bash
ls /Applications/
```

**2. Analyze each app and suggest a strategy**

For each app, suggest one of:
- ✅ **Install fresh** — app is available via brew cask or direct download, no important local data
- ⚠️ **Copy app + data** — has saved connections, collections, or settings worth preserving
- 🔄 **Data only** — reinstall fresh but copy `~/Library/Application Support/<App>` for sessions/history
- ❌ **Skip** — already installed on new Mac, App Store app (re-download), or not worth transferring

**3. Ask the user to confirm the strategy for each app**
Go through the list one by one. The user can override any suggestion.

**4. For apps marked "Install fresh"**
Generate a single `brew install --cask` command for all of them.

**5. For apps marked "Copy"**
Use rsync to copy the `.app` bundle:
```bash
rsync -avh --progress \
  -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  "/Applications/AppName.app" \
  NEW_USER@NEW_MAC_IP:/Applications/
```

**6. Copy app data**
For every app (whether copied or freshly installed), check for data in:
- `~/Library/Application Support/<App>`
- `~/Library/Containers/<bundle-id>`
- `~/Library/Preferences/<bundle-id>.plist`
- `~/Library/<App>` (some apps use this)

Use tar pipe for paths with spaces:
```bash
cd ~/Library/Application\ Support && \
tar czf - AppName | \
ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes NEW_USER@NEW_MAC_IP \
"cd ~/Library/Application\ Support && tar xzf -"
```

**7. Known app data locations**
Common apps and their data:
- Chrome: `~/Library/Application Support/Google/Chrome` (exclude Cache, Code Cache, GPUCache)
- VS Code: `~/Library/Application Support/Code/User/History` (settings sync via GitHub, only history needs copying)
- Cursor: `~/Library/Application Support/Cursor`
- Postico: `~/Library/Containers/at.eggerapps.Postico`
- DBeaver: `~/Library/DBeaverData`
- TablePlus: `~/Library/Application Support/com.tinyapp.TablePlus`
- Insomnia: `~/Library/Application Support/Insomnia`
- Sequel Pro: `~/Library/Application Support/Sequel Pro`
- iTerm2: `~/Library/Application Support/iTerm2`
- Figma: `~/Library/Application Support/Figma`
- Sketch: `~/Library/Containers/com.bohemiancoding.sketch3`

---

## Phase 7: Copy Dotfiles

### Steps:

**1. SSH keys**
```bash
rsync -avh -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/.ssh/ NEW_USER@NEW_MAC_IP:~/.ssh/
# Fix permissions on new Mac:
ssh NEW_USER@NEW_MAC_IP "chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_rsa* ~/.ssh/id_ed25519* 2>/dev/null"
```

**2. Git config**
```bash
rsync -avh -e "ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes" \
  ~/.gitconfig NEW_USER@NEW_MAC_IP:~/.gitconfig
```

**3. Shell config**
Copy `.zshrc`, `.zprofile`, `.zshenv` (or `.bashrc`/`.bash_profile` for bash, `~/.config/fish/` for fish).

After copying, scan for hardcoded old username paths and fix them:
```bash
sed -i '' 's|/Users/OLD_USER|/Users/NEW_USER|g' ~/.zshrc ~/.zprofile
```

**4. Shell history (cleaned)**
Before copying, offer to clean the history:
- Remove duplicate consecutive commands
- Remove short gibberish entries (length < 3)
- Optionally scan for accidentally typed secrets (passwords, tokens)

```bash
# Preview duplicates and gibberish before cleaning
cat ~/.zsh_history | sed 's/^:[^;]*;//' | awk 'length($0) <= 2' | sort | uniq -c | sort -rn | head -20
```

**5. GPG keys** (if detected)
```bash
gpg --export-secret-keys --armor > ~/gpg-backup.asc
# Transfer and import on new Mac:
gpg --import ~/gpg-backup.asc
```

**6. Other dotfiles**
Ask: "Do you have any other dotfiles to copy? (e.g. `~/.npmrc`, `~/.gemrc`, `~/.aws/`, `~/.kube/`)"

---

## Phase 8: Migrate Claude Code Data

### Steps:

**1. Check what exists**
```bash
ls -la ~/.claude/
```

**2. Copy all Claude Code data** (excluding `ide/` lock files and `cache/`):
```bash
tar czf - -C ~/ \
  --exclude=".claude/ide" \
  --exclude=".claude/cache" \
  .claude | \
ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes NEW_USER@NEW_MAC_IP \
"cd ~/ && tar xzf -"
```

**3. Fix hardcoded paths in files** (critical if username changed):
```bash
ssh NEW_USER@NEW_MAC_IP "grep -rl 'OLD_USER' ~/.claude/ | xargs sed -i '' 's|OLD_USER|NEW_USER|g' 2>/dev/null"
```

**4. Rename project directories** (they encode the absolute path):
```bash
ssh NEW_USER@NEW_MAC_IP "python3 -c \"
import os
projects_dir = os.path.expanduser('~/.claude/projects')
for name in os.listdir(projects_dir):
    if 'OLD_USER' in name:
        new_name = name.replace('OLD_USER', 'NEW_USER')
        os.rename(os.path.join(projects_dir, name), os.path.join(projects_dir, new_name))
        print(f'Renamed: {name} -> {new_name}')
\""
```

**5. Verify**
```bash
ssh NEW_USER@NEW_MAC_IP "grep -rl 'OLD_USER' ~/.claude/ 2>/dev/null | wc -l"
# Should return 0
```

**6. Remind the user** to run `claude login` on the new Mac to authenticate.

---

## Phase 9: Optimize Shell (only if needed)

**Only suggest this phase if:**
- Shell startup takes more than 1 second (ask the user: "Does your terminal feel slow to open?")
- OR heavy loaders are detected in `.zshrc`: `nvm`, `pyenv`, `rbenv`, `rvm`, `jenv`, `sdkman`, `asdf`

Read `references/shell-optimization.md` for lazy-load patterns.

### Steps:
1. Measure current startup time: `time zsh -i -c exit`
2. Identify slow loaders in `.zshrc`
3. Offer lazy-load replacements for each detected loader
4. Apply only the ones the user agrees to
5. Measure again to confirm improvement

---

## Phase 10: Re-authenticate

Generate a checklist based on CLIs detected in Phase 1:

```
Re-authentication checklist:
[ ] Claude Code:   claude login
[ ] AWS:           aws configure  (or: aws sso login)
[ ] Google Cloud:  gcloud auth login && gcloud auth application-default login
[ ] Fly.io:        flyctl auth login
[ ] Stripe:        stripe login
[ ] GitHub CLI:    gh auth login
[ ] Heroku:        heroku login
[ ] Vercel:        vercel login
[ ] Netlify:       netlify login
[ ] npm (private): npm login
[ ] Docker Hub:    docker login
[ ] ngrok:         ngrok config add-authtoken YOUR_TOKEN
```

Also remind:
- Check `.env` files in projects — some may have machine-specific paths
- Re-activate any software licenses (JetBrains, Adobe, Sketch, etc.) that are machine-locked
- Test a project end-to-end to confirm the migration worked

---

## Phase 11: Verify Migration

Run this phase on the **new Mac** to confirm everything migrated correctly.
Compare results against the Phase 1 audit from the old Mac.

### Steps:

**1. Tool versions**
```bash
echo "=== Node ===" && node --version 2>/dev/null
echo "=== Python ===" && python3 --version 2>/dev/null
echo "=== Ruby ===" && ruby --version 2>/dev/null
echo "=== Go ===" && go version 2>/dev/null
echo "=== Rust ===" && rustc --version 2>/dev/null
echo "=== Java ===" && java --version 2>/dev/null
echo "=== PHP ===" && php --version 2>/dev/null
echo "=== Brew packages ===" && brew leaves 2>/dev/null | wc -l
```
Compare versions and package counts against Phase 1 output.

**2. Services running**
```bash
brew services list
```
All services that were running on old Mac should show `started`.

**3. Databases**
```bash
# PostgreSQL
psql -U $(whoami) -l 2>/dev/null | grep -v "template\|postgres\|^-\|^List\|^Name\|^(" | wc -l

# MongoDB
mongosh --quiet --eval "db.adminCommand({listDatabases:1}).databases.map(d => d.name)" 2>/dev/null

# MySQL
mysql -u root -e "SHOW DATABASES;" 2>/dev/null | grep -v "information_schema\|performance_schema\|sys\|mysql\|Database"

# Redis
redis-cli ping 2>/dev/null
```
Compare database counts against what was dumped in Phase 5.

**4. Projects**
```bash
# Count git repos (run for each directory in PROJECT_DIRS)
find $PROJECT_DIRS -maxdepth 4 -name ".git" -type d 2>/dev/null | wc -l

# Total size
du -sh $PROJECT_DIRS
```
Compare against old Mac counts.

**5. SSH keys**
```bash
ls ~/.ssh/
ssh-add -l 2>/dev/null
```
All key pairs should be present. Test a real SSH connection:
```bash
ssh -T git@github.com 2>&1
```

**6. Shell startup time**
```bash
time zsh -i -c exit
```
Should be under 1 second with lazy loading applied.

**7. Claude Code**
```bash
ls ~/.claude/projects/ | wc -l
```
Project count should match old Mac.

**8. Generate a verification report**

After running all checks, produce a summary report:

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

Flag any mismatches with ❌ and suggest fixes.
