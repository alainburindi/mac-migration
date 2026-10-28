# Transfer Files — Phase 4

All commands run on the **old Mac**, pushing to the new Mac.

## 1. Connect the two Macs

Ask: "How do you want to connect the two Macs?"
- **Option A — Wi-Fi** (same network): get the **new Mac's** IP from `System Settings > Wi-Fi > Details` on the new Mac.
- **Option B — Thunderbolt/USB-C cable** (fastest): connect them directly, then read the **new Mac's** IP under `System Settings > Network > Thunderbolt Bridge`. A self-assigned `169.254.x.x` address is fine. The cable must support Thunderbolt (look for the ⚡ symbol); charge-only cables won't work.

On the **new Mac**, enable `System Settings > General > Sharing > Remote Login`. Click (i) and turn on **"Allow full disk access for remote users"**. Without it, copies into `~/Library/Containers`, `~/Desktop`, `~/Documents` and `~/Downloads` fail with "Operation not permitted".

On the **old Mac**, give the terminal app running Claude Code Full Disk Access (`System Settings > Privacy & Security > Full Disk Access`). Without it, protected folders can't be read.

## 2. Migration SSH key and `newmac` alias

Use a dedicated key, so no assumptions about the user's existing keys (`id_rsa` vs `id_ed25519`) are needed and cleanup is trivial. All later phases use the `newmac` alias.

Create the key if it doesn't exist:
```bash
[ -f ~/.ssh/mac_migration_ed25519 ] || ssh-keygen -t ed25519 -N "" -C "mac-migration" -f ~/.ssh/mac_migration_ed25519
```

Add the alias block at the **top** of `~/.ssh/config`:
- SSH uses the first value it finds for each option, so a `Host *` block above ours would override `User`.
- The trailing `Host *` line closes our block, so the user's top-level settings (`Include`, `UseKeychain`, `IdentityAgent`, …) keep applying to all hosts.
- The command is safe to re-run: it does nothing if the block is already there, and never overwrites an existing backup.

Show it to the user and confirm before running:
```bash
mkdir -p ~/.ssh && touch ~/.ssh/config
if ! /usr/bin/grep -q '^# >>> mac-migration' ~/.ssh/config; then
  [ -f ~/.ssh/config.pre-migration ] || cp ~/.ssh/config ~/.ssh/config.pre-migration
  { cat <<'EOF'
# >>> mac-migration
Host newmac
  HostName NEW_MAC_IP
  User NEW_USER
  IdentityFile ~/.ssh/mac_migration_ed25519
  IdentitiesOnly yes
Host *
# <<< mac-migration

EOF
  cat ~/.ssh/config; } > ~/.ssh/config.tmp && mv ~/.ssh/config.tmp ~/.ssh/config && chmod 600 ~/.ssh/config
fi
```
If the IP changes later (e.g. switching from Wi-Fi to cable), edit the `HostName` line inside the block.

Install the public key without a password prompt. Print the line:
```bash
echo "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '$(cat ~/.ssh/mac_migration_ed25519.pub)' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
Ask the user to paste that line into Terminal **on the new Mac**.

Then test from the old Mac. `accept-new` records the host key on first contact without a yes/no prompt:
```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 newmac 'echo connected; sw_vers -productVersion; uname -m'
```

## 3. Send the state file and setup script, then run the setup script

```bash
rsync -a ~/mac-migration-state.md ~/new-mac-setup.sh newmac:
```

Tell the user to run it **in Terminal on the new Mac**. It needs their password for Homebrew and the Xcode Command Line Tools:
```
zsh ~/new-mac-setup.sh
```
It can run while the project transfer below is in progress, but it must finish before Phase 5.

## 4. Before copying projects

**Destination sync.** Work Macs often sync `~/Documents` or `~/Desktop` to iCloud or OneDrive. Copying git repos into a synced folder causes evicted objects and duplicate files (e.g. "file 2"). Check on the new Mac:
```bash
ssh newmac 'ls ~/Library/CloudStorage 2>/dev/null; ls -d ~/Library/Mobile\ Documents/com~apple~CloudDocs/Desktop ~/Library/Mobile\ Documents/com~apple~CloudDocs/Documents 2>/dev/null'
```
Then ask the user directly, because sync settings aren't always visible this way. If a destination is synced, choose an unsynced folder such as `~/code` and tell the user that paths will change.

**Stop writers.** Quit apps, dev servers (`npm run dev`, Rails, Django …) and Docker containers that write into the project folders. SQLite databases and other files copied mid-write end up corrupt.
```bash
docker ps --format '{{.Names}}' 2>/dev/null
```

**Username warning.** If usernames differ, warn: "Your dotfiles and configs contain hardcoded paths with your old username. We'll fix these in Phase 7 and Phase 8."

## 5. Project directories

For each project directory, ask which mode to use.

### Mode A — Full copy (default)

Keeps unpushed commits, stashes, local branches, uncommitted work and gitignored credential files (`.env` etc.).

Exclude rebuildable folders:
- They contain binaries built for the old Mac's architecture (native node modules) and absolute paths to the old user's interpreters (Python virtualenvs), so copying them produces broken environments.
- Phase 10 rebuilds them.
- Show the default list and let the user adjust it. Also ask about `target/` (Rust), `build/`, `dist/` and `Pods/`, since some repos commit these.

Run it in the background with a log (see Long-running commands in SKILL.md):
```bash
mkdir -p ~/mac-migration-logs
caffeinate -dims rsync -a --partial --stats \
  --exclude=node_modules/ --exclude=.venv/ --exclude=venv/ --exclude=__pycache__/ \
  --exclude=.next/ --exclude=.turbo/ --exclude=.parcel-cache/ \
  --exclude=.pytest_cache/ --exclude=.mypy_cache/ \
  "/Users/OLD_USER/code/" "newmac:code/" > ~/mac-migration-logs/rsync-code.log 2>&1
echo "exit: $?"
```
- The destination path is relative to the new Mac's home, so `~/code` maps to `/Users/NEW_USER/code` even when usernames differ.
- Keep the trailing slash on the source; without it, rsync nests the directory one level deeper.
- When it finishes, check `exit: 0` and `tail -20 ~/mac-migration-logs/rsync-code.log`. The `--stats` summary shows how many files were transferred.

### Mode B — Fresh clone + credentials only

For users who prefer to `git clone` everything again. Only offer it for repos with nothing unpushed (Phase 2).

First list the gitignored credential files and confirm them with the user:
```bash
find "/Users/OLD_USER/code" \
  \( -name node_modules -o -name .venv -o -name venv -o -name .git -o -name vendor \) -prune -o \
  -type f \( -name ".env" -o -name ".env.*" -o -name "credentials.json" -o -name "*.pem" -o -name "*.key" -o -name "*.p12" -o -name "*.secret" \) -print \
  2>/dev/null | sort
```
Then copy only those files. Excludes must come before `--include='*/'`, and `-m` skips directories that end up empty:
```bash
rsync -a -m \
  --exclude=.git/ --exclude=node_modules/ --exclude=.venv/ --exclude=venv/ --exclude=vendor/ \
  --include='*/' \
  --include='.env' --include='.env.*' --include='credentials.json' \
  --include='*.pem' --include='*.key' --include='*.p12' --include='*.secret' \
  --exclude='*' \
  "/Users/OLD_USER/code/" "newmac:code/"
```

## 6. Other home folders

Ask which of these to transfer: `~/Desktop`, `~/Documents` (if not already covered as a project directory), `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music`.
- Skip any folder synced by iCloud on the old Mac ("Desktop & Documents Folders" enabled); it syncs on its own.
- For `~/Pictures/Photos Library.photoslibrary`, prefer iCloud Photos. If copying, quit Photos on both Macs first.

Use the same background pattern as Mode A:
```bash
caffeinate -dims rsync -a --partial --stats "/Users/OLD_USER/Desktop/" "newmac:Desktop/" > ~/mac-migration-logs/rsync-desktop.log 2>&1
echo "exit: $?"
```

## 7. Resumability

Remind the user: "rsync is safe to re-run. It skips files already transferred, and `--partial` resumes large files that were interrupted. If a transfer stops, run the same command again."

Update the state file: mark Phase 4 done, record the transfer mode, destination and excludes per directory, and record project directory sizes (`du -sh`).
