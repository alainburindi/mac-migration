# Copy Apps and App Data — Phase 6

## 1. List installed apps

```bash
ls /Applications/ ~/Applications/ 2>/dev/null
brew list --cask 2>/dev/null                      # already handled by the setup script
mdfind "kMDItemAppStoreHasReceipt == 1" -onlyin /Applications 2>/dev/null   # App Store apps
```

For non-brew apps, check whether each one is Intel-only (it would need Rosetta on Apple Silicon; prefer reinstalling a native build):
```bash
for app in /Applications/*.app; do
  bin="$app/Contents/MacOS/$(defaults read "$app/Contents/Info" CFBundleExecutable 2>/dev/null)"
  [ -f "$bin" ] && echo "$(basename "$app"): $(lipo -archs "$bin" 2>/dev/null)"
done
```

## 2. Suggest a strategy per app

For each app, suggest one of:
- ✅ **Install fresh:** available via brew cask or direct download, no important local data. Intel-only apps belong here when a native build exists.
- ⚠️ **Copy app + data:** not reinstallable (old version, no installer), or has saved connections or settings worth preserving.
- 🔄 **Data only:** reinstall fresh, then copy the app's data folder for sessions and history.
- ❌ **Skip:** already on the new Mac, an App Store app (re-download it), or not worth transferring.

## 3. Confirm with the user

Go through the list one by one. The user can override any suggestion.

## 4. Install fresh

Generate a single `brew install --cask` command for these apps. The user runs it on the new Mac, or Claude runs it over `ssh newmac` with Homebrew loaded.

## 5. Copy app bundles

Use `--extended-attributes` so code-signing and quarantine metadata come along; without it some apps report as "damaged". `/Applications` isn't writable for standard (non-admin) users, which is common on managed work Macs. Check first:
```bash
ssh newmac 'test -w /Applications && echo writable || echo "not writable — use ~/Applications"'
```
```bash
rsync -a --extended-attributes "/Applications/AppName.app" newmac:/Applications/
# or, if not writable:
rsync -a --extended-attributes "/Applications/AppName.app" newmac:Applications/
```

## 6. Copy app data

**Quit each app on both Macs before copying its data.** Apps keep SQLite databases and caches open, and copying them while the app runs can corrupt them.

Check each app for data in:
- `~/Library/Application Support/<App>`
- `~/Library/Containers/<bundle-id>`
- `~/Library/Group Containers/<team-id>.<group>`
- `~/Library/Preferences/<bundle-id>.plist`
- `~/Library/<App>` (some apps use this)

`Containers` and `Group Containers` are protected. Copying them needs Full Disk Access for the terminal on the old Mac, and "Allow full disk access for remote users" on the new Mac (both set up in Phase 4).

Use a tar pipe, which handles spaces in paths and creates the parent folder:
```bash
tar czf - -C ~/Library/"Application Support" "AppName" | \
  ssh newmac 'mkdir -p ~/Library/"Application Support" && tar xzf - -C ~/Library/"Application Support"'
```

Preferences are cached by `cfprefsd`. After copying `.plist` files, run `killall cfprefsd` on the new Mac, before opening the app, so the copied files are picked up instead of being overwritten.

## 7. Known app data locations

- **Chrome:** `~/Library/Application Support/Google/Chrome`.
  - Cookies, logged-in sessions and saved passwords are encrypted with a key in the old Mac's Keychain and **will not decrypt on the new Mac**. Recommend Chrome Sync (signed-in profile) instead.
  - Copy the folder only for unsynced profiles, extensions' local data or history, and exclude caches:
    ```bash
    tar czf - -C ~/Library/"Application Support"/Google \
      --exclude='Cache' --exclude='Code Cache' --exclude='GPUCache' --exclude='Service Worker' Chrome | \
      ssh newmac 'mkdir -p ~/Library/"Application Support"/Google && tar xzf - -C ~/Library/"Application Support"/Google'
    ```
- **VS Code:**
  - Settings and keybindings come back through Settings Sync (GitHub/Microsoft sign-in).
  - Copy `~/Library/Application Support/Code/User/` (settings, snippets, local history) if Sync isn't used.
  - Extensions:
    ```bash
    code --list-extensions > ~/vscode-extensions.txt
    rsync -a ~/vscode-extensions.txt newmac:
    # On the new Mac:
    xargs -n1 code --install-extension < ~/vscode-extensions.txt
    ```
- **Cursor:** `~/Library/Application Support/Cursor` (extensions list via `cursor --list-extensions`).
- **Postico:** `~/Library/Containers/at.eggerapps.Postico`
- **DBeaver:** `~/Library/DBeaverData`. Saved passwords are stored in the macOS Keychain; re-enter them.
- **TablePlus:** `~/Library/Application Support/com.tinyapp.TablePlus`. Passwords are in the Keychain.
- **Insomnia:** `~/Library/Application Support/Insomnia`
- **Sequel Pro / Sequel Ace:** `~/Library/Application Support/Sequel Pro`, `~/Library/Containers/com.sequel-ace.sequel-ace`
- **iTerm2:** `~/Library/Preferences/com.googlecode.iterm2.plist` plus `~/Library/Application Support/iTerm2`
- **Figma:** `~/Library/Application Support/Figma` (mostly cloud; usually only fonts and settings).
- **Sketch:** `~/Library/Containers/com.bohemiancoding.sketch3`

General rule: anything an app keeps in the Keychain (passwords, tokens, license keys) does not move with its data folder. Tell the user to expect to sign in again.

Update the state file when done.
