# Copy Dotfiles — Phase 7

## 1. SSH keys

First check what the new Mac already has. Work Macs often come with IT-issued keys:
```bash
ssh newmac 'ls -la ~/.ssh/'
```
If any key filenames exist on both Macs, ask the user which to keep before copying. If both are needed, rename one side (e.g. `id_ed25519_personal`).

Rules:
- Never overwrite `authorized_keys` on the new Mac: it holds the migration key from Phase 4, and replacing it breaks passwordless SSH for the rest of the migration.
- `--ignore-existing` protects keys already on the new Mac.
- Don't copy the migration key, or the `config.pre-migration` backup.
- Merge `known_hosts` instead of replacing it.

```bash
rsync -a --ignore-existing \
  --exclude=authorized_keys --exclude=known_hosts \
  --exclude='mac_migration_ed25519*' --exclude=config.pre-migration --exclude=config.tmp \
  ~/.ssh/ newmac:.ssh/
ssh newmac 'cat >> ~/.ssh/known_hosts' < ~/.ssh/known_hosts
# Fix permissions on new Mac (private keys and config 600, public keys untouched)
ssh newmac "chmod 700 ~/.ssh && find ~/.ssh -type f ! -name '*.pub' -exec chmod 600 {} +"
```

If `~/.ssh/config` was copied, it includes the `newmac` alias block from Phase 4, which is meaningless on the new Mac. Remove it there:
```bash
ssh newmac "sed -i '' '/^# >>> mac-migration/,/^# <<< mac-migration/d' ~/.ssh/config"
```

## 2. Symlinked dotfiles

Users of GNU stow, chezmoi, yadm or a dotfiles repo often have `~/.zshrc` and friends as symlinks into that repo. A copied symlink points at `/Users/OLD_USER/...`, which doesn't exist on the new Mac. Check:
```bash
for f in ~/.zshrc ~/.zprofile ~/.zshenv ~/.bashrc ~/.bash_profile ~/.gitconfig ~/.vimrc ~/.tmux.conf; do
  [ -L "$f" ] && echo "$f -> $(readlink "$f")"
done
find ~/.config -maxdepth 1 -type l -exec sh -c 'echo "$1 -> $(readlink "$1")"' _ {} \; 2>/dev/null
```
If any show up:
- **The repo was already copied in Phase 4:** re-create the links on the new Mac with the repo's own tool (`stow`, `chezmoi apply`, `yadm`), or with `ln -sf` pointing at the new-username path.
- **Otherwise:** ask whether to copy the dotfiles repo, or to copy the resolved files instead. For the resolved files, use `rsync -L`, which copies the file the link points to.

## 3. Git config

Check whether the new Mac already has a `~/.gitconfig` (e.g. a work identity):
```bash
ssh newmac 'cat ~/.gitconfig 2>/dev/null'
```
If it does, show both files and ask the user whether to replace it, merge them, or keep the new one. **Run the copy only if the user chose replace.** For merge, write the merged file and show it first.
```bash
rsync -a ~/.gitconfig newmac:
```
Also copy the files the config references, if they exist (`core.excludesfile`, `core.hooksPath`, `include.path`, `commit.template`). List them:
```bash
git config --global --get-regexp '^(core\.excludesfile|core\.hookspath|include\.path|commit\.template)$'
```
Copy each one to the same path relative to home, e.g. `rsync -a ~/.gitignore_global newmac:` or `rsync -a ~/.githooks/ newmac:.githooks/`.

## 4. Shell config

Find the files `.zshrc` loads, so they come along too (`~/.p10k.zsh`, `~/.aliases`, `~/.zsh/`, …). Without them the shell breaks, or the p10k setup wizard starts again:
```bash
/usr/bin/grep -nE '^\s*(source|\.)\s' ~/.zshrc ~/.zprofile 2>/dev/null
```

Copy `.zshrc`, `.zprofile`, `.zshenv` and the files found above (or `.bashrc`/`.bash_profile` for bash, `~/.config/fish/` for fish):
- The setup script doesn't write `.zshrc`.
- The oh-my-zsh installer creates a template `.zshrc` when none exists. Overwriting that template is expected, so no diff is needed.
- Anything else already in `.zshrc` on the new Mac: show a diff first.

```bash
for f in .zshrc .zprofile .zshenv .p10k.zsh; do [ -f ~/$f ] && rsync -a ~/$f newmac:; done
```

For oh-my-zsh users, also copy custom plugins and themes (the setup script installed oh-my-zsh itself):
```bash
rsync -a ~/.oh-my-zsh/custom/ newmac:.oh-my-zsh/custom/
```

**Homebrew on PATH (Intel → Apple Silicon).** On Intel, `/usr/local/bin` is on the default PATH, so the old `.zprofile` often has no `brew shellenv` line. Without one, new Terminals on the new Mac (and the Phase 9–11 Claude session) can't find `brew`, `psql` or anything else installed with Homebrew. The setup script adds the line, but the copy above may have replaced `.zprofile`, so re-check:
```bash
ssh newmac 'bash -s' <<'EOF'
line='eval "$(/opt/homebrew/bin/brew shellenv)"'
/usr/bin/grep -qF "$line" ~/.zprofile 2>/dev/null || echo "$line" >> ~/.zprofile
tail -3 ~/.zprofile
EOF
```
Also point out Intel-only paths in the copied config, like `/usr/local/bin/brew` or `/usr/local/opt/...`. On Apple Silicon these live under `/opt/homebrew`.

## 5. Fix old-username paths (only if usernames differ)

Run this **on the new Mac**, after everything in this phase has been copied. Running `sed` locally would rewrite the old Mac's files.

Find every copied config file that still mentions the old home folder:
- shell files, `.gitconfig`, `.ssh/config`
- `.npmrc` (`prefix=` breaks npm globals), `.aws/`, `.kube/` (certificate paths), `.config/`, oh-my-zsh custom
- `crontab.txt` and the copied LaunchAgents

```bash
ssh newmac 'bash -s' <<'EOF'
cd ~ && /usr/bin/grep -rlI '/Users/OLD_USER' \
  .zshrc .zprofile .zshenv .bashrc .bash_profile .p10k.zsh .gitconfig .npmrc .gemrc .ssh/config \
  .aws .kube .docker/config.json .config .oh-my-zsh/custom crontab.txt Library/LaunchAgents 2>/dev/null
EOF
```

Show the list and confirm with the user, then rewrite those files. If `OLD_USER` contains a `.`, escape it as `\.` in the `sed` and `grep` patterns below. The `sed` replaces `/Users/OLD_USER` when it's followed by any character that can't be part of a username (`/`, `"`, `'`, `:`, space, …) or ends the line, so `/Users/OLD_USERfoo` is left alone.

LaunchAgent plists may be binary, and `sed` would corrupt them. Convert them to XML first:
```bash
ssh newmac 'bash -s' <<'EOF'
cd ~
for f in FILES_FROM_LIST; do
  case "$f" in *.plist) plutil -convert xml1 "$f" ;; esac
  sed -i '' -E 's#/Users/OLD_USER([^A-Za-z0-9._-])#/Users/NEW_USER\1#g; s#/Users/OLD_USER$#/Users/NEW_USER#g' "$f"
done
# Should print nothing
/usr/bin/grep -rlI -E '/Users/OLD_USER([^A-Za-z0-9._-]|$)' FILES_FROM_LIST
true
EOF
```

## 6. Shell history

Before copying, offer to clean the history:
- remove duplicate consecutive commands
- remove short gibberish entries (fewer than 3 characters)
- optionally scan for accidentally typed secrets

Preview the gibberish entries first:
```bash
sed 's/^:[^;]*;//' ~/.zsh_history | awk 'length($0) <= 2' | sort | uniq -c | sort -rn | head -20
```

For the secret scan, never print the matches themselves; they would land in the conversation transcript. Show only line numbers and the first word of each line:
```bash
sed 's/^:[^;]*;//' ~/.zsh_history | /usr/bin/grep -niE 'token|secret|passw|api_?key|bearer|://[^ /]*:[^ /]*@' | awk -F: '{split($2, w, " "); print "line " $1 ": " w[1] " …"}'
```
Write any cleaned version to a new file and copy that; never edit `~/.zsh_history` on the old Mac in place.
```bash
rsync -a ~/.zsh_history newmac:
```

## 7. GPG keys (if detected)

Pipe the keys straight to the new Mac, so no unencrypted secret key ever touches the disk.

Exporting asks for each key's passphrase. On macOS with `pinentry-mac`, a dialog appears and the pipe works from the Bash tool. With a terminal-only pinentry, ask the user to run the export pipe in their own Terminal window on the old Mac.
```bash
gpg --list-secret-keys --keyid-format long
gpg --export-secret-keys --armor | ssh newmac 'eval "$(/opt/homebrew/bin/brew shellenv)"; gpg --batch --import'
gpg --export-ownertrust | ssh newmac 'eval "$(/opt/homebrew/bin/brew shellenv)"; gpg --import-ownertrust'
ssh newmac 'eval "$(/opt/homebrew/bin/brew shellenv)"; gpg --list-secret-keys --keyid-format long'
```
If an earlier attempt left a file like `~/gpg-backup.asc`, delete it on both Macs with `rm -P`.

## 8. Other dotfiles and user config

Ask about each one detected in the audit, and copy it before running section 5:
- **Tool config:**
  - `~/.npmrc` (may contain registry tokens), `~/.gemrc`, `~/.config/gh/`
  - `~/.aws/`, `~/.kube/`, `~/.docker/config.json`
  - `~/.config/` in general, which many CLIs use
- **Fonts:** `rsync -a ~/Library/Fonts/ newmac:Library/Fonts/`

**Scheduled jobs**

`crontab`. Show the copied file to the user after the path fix in section 5, and install it only once they confirm:
```bash
crontab -l > ~/crontab.txt 2>/dev/null && rsync -a ~/crontab.txt newmac:
ssh newmac 'cat ~/crontab.txt'
ssh newmac 'crontab ~/crontab.txt'
```

LaunchAgents: copy only the user's own agents from `~/Library/LaunchAgents/`, not the ones apps install (`com.google.keystone.*`, `com.adobe.*`, etc.):
```bash
rsync -a ~/Library/LaunchAgents/NAME.plist newmac:Library/LaunchAgents/
```
They load at next login, or immediately when the user runs `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/NAME.plist` in Terminal on the new Mac.

**`/etc/hosts`**

Show the custom entries:
```bash
/usr/bin/grep -vE '^\s*(#|$)|localhost|broadcasthost' /etc/hosts
```
Editing `/etc/hosts` needs `sudo`, so the user adds them on the new Mac themselves (`sudo nano /etc/hosts` in Terminal).

Update the state file: record the number of SSH key pairs copied.
