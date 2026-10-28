---
name: mac-migration
description: Guide the user through migrating their complete developer environment from an old Mac to a new Mac without Migration Assistant. Use this when the user says "migrate my Mac", "setting up a new Mac", "new MacBook", "transferring to new Mac", "Mac migration", mentions a different username on a work Mac, or mentions moving dev tools, projects, databases or dotfiles between Macs.
argument-hint: "[audit|git-check|generate-script|transfer|databases|apps|dotfiles|claude-data|shell|auth|verify]"
---

# Mac Migration

Help the user migrate their complete developer setup from an old Mac to a new Mac **without using Migration Assistant**.

This skill is specifically designed for situations where Migration Assistant is not an option:
- **Different usernames** between old and new Mac (e.g. personal vs work machine)
- **Work/corporate machines** with assigned usernames or MDM restrictions
- **Wanting full control** over exactly what gets migrated

This is an 11-phase interactive process. Work through each phase conversationally:
run checks, show what was found, confirm with the user, then generate outputs.

## Ground rules

**Where things run**
- Phases 1–8 run in a Claude Code session on the **old Mac**. Commands for the new Mac go over SSH through the `newmac` alias set up in Phase 4.
- Phases 9–11 run in a Claude Code session on the **new Mac** (Phase 8 copies Claude Code data, including this plugin, so `/alb:mac-migration shell` works there).

**Permissions**
- Run read-only commands (audit, checks, scans, listings) directly with the Bash tool without asking first.
- Before any command that writes, copies, overwrites or deletes, show it and get the user's confirmation.
- The Bash tool has no terminal, so it can't answer prompts. Don't rely on the `!` prefix for prompts either. Anything that asks for a password or passphrase (`sudo`, `mysql -p`, a terminal GPG pinentry) goes to the user to run in their own Terminal window. Prefer designs that need no prompt at all (e.g. Phase 4 installs the SSH key by pasting a line on the new Mac).

**Long-running commands**
The Bash tool times out after a few minutes, which kills big rsyncs and database dumps partway through. For anything that may take more than about a minute:
- Run it with `run_in_background`.
- Prefix it with `caffeinate -dims` so the Mac doesn't sleep.
- Log to `~/mac-migration-logs/<step>.log` instead of the conversation (`mkdir -p ~/mac-migration-logs` first).
- For rsync, use `-a --partial` without `-v`/`--progress`, so per-file output doesn't flood the context and interrupted large files resume.
- When it finishes, check the exit code and `tail` the log.

**Placeholders**
- Replace `OLD_USER`, `NEW_USER`, `NEW_MAC_IP` and `PG_FORMULA` (e.g. `postgresql@16`) with the real values before running any command.
- Project directories (`PROJECT_DIRS`) are always written as quoted absolute paths, e.g. `"/Users/john/code" "/Users/john/work"`. Never put them in a shell variable: zsh does not word-split variables, and `~` does not expand inside quotes.

**Homebrew paths**
- On the old Mac, never hardcode `/opt/homebrew` — Intel Macs use `/usr/local`. Use `$(brew --prefix)`.
- Non-interactive SSH sessions don't load the Homebrew PATH. Any `ssh newmac` command that uses a brew-installed tool must start with `eval "$(/opt/homebrew/bin/brew shellenv)"`. Use the heredoc form to avoid quoting problems:
  ```bash
  ssh newmac 'bash -s' <<'EOF'
  eval "$(/opt/homebrew/bin/brew shellenv)"
  psql --version
  EOF
  ```

**State file**
Conversation memory does not survive a new session or the move to the new Mac. Keep `~/mac-migration-state.md` as the single source of truth:
```markdown
# Mac Migration State
- Old Mac user: OLD_USER (arch: x86_64|arm64)
- New Mac user: NEW_USER (arch: arm64)
- PostgreSQL formula: postgresql@NN (from the audit; used as PG_FORMULA)
- New Mac IP: NEW_MAC_IP (SSH alias: newmac, key: ~/.ssh/mac_migration_ed25519)
- Project dirs: "/Users/OLD_USER/code" "/Users/OLD_USER/work"

## Phases
- [ ] 1 Audit  - [ ] 2 Git check  - [ ] 3 Setup script  - [ ] 4 Transfer
- [ ] 5 Databases  - [ ] 6 Apps  - [ ] 7 Dotfiles  - [ ] 8 Claude Code
- [ ] 9 Shell  - [ ] 10 Re-auth  - [ ] 11 Verify

## Audit summary
(runtimes + versions, brew counts, databases, CLIs, shell framework)

## Counts for verification
(git repos, project dir sizes, databases per engine, SSH key pairs, Claude projects)
```
- Create it after the welcome questions. Update it at the end of every phase.
- When invoked with an argument, or when resuming, read it first. If it's missing, ask the welcome questions again.

## Arguments

If invoked with `$ARGUMENTS`, read the state file, then jump to the corresponding phase:
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
Mac Migration — 11 Phases

On the old Mac:
[ ] Phase 1:  Audit old Mac
[ ] Phase 2:  Check unpushed git work
[ ] Phase 3:  Generate setup script
[ ] Phase 4:  Transfer files (+ run setup script on new Mac)
[ ] Phase 5:  Migrate databases
[ ] Phase 6:  Copy apps and data
[ ] Phase 7:  Copy dotfiles
[ ] Phase 8:  Migrate Claude Code data

On the new Mac:
[ ] Phase 9:  Optimize shell (if needed)
[ ] Phase 10: Re-authenticate and rebuild dependencies
[ ] Phase 11: Verify migration and clean up
```

If `~/mac-migration-state.md` exists, show its progress and offer to resume. Otherwise ask:
1. "What is your username on the **old** Mac?" (default: output of `whoami`)
2. "What is your username on the **new** Mac?" (e.g. `johnsmith`). If they differ, note that later phases will fix hardcoded paths.
3. "Where do you store your projects?" (e.g. `~/Documents/work`, `~/Projects`, `~/code`, or several). Convert the answer to quoted absolute paths.

Create the state file with these answers.

---

## Phase 1: Audit Old Mac

1. Read `references/audit-commands.md` and run each block directly with the Bash tool. Don't ask the user to run them; they're read-only.
2. Summarize the results in a table:
   - Languages and runtimes (with versions)
   - Package managers and their global packages
   - Databases (Homebrew and Docker)
   - Cloud CLIs
   - Shell and framework
   - SSH/GPG keys
   - Scheduled jobs (crontab, LaunchAgents)
3. Ask: "Does anything look missing or wrong?"
4. Write the summary to the state file. Phase 3 builds the setup script from it, and Phase 11 compares against it.

Things to detect:
- **Language runtimes:** Node (nvm/fnm/volta/brew), Python (pyenv/brew/uv/conda), Ruby (rbenv/rvm/brew), Go, Rust, Java (jenv/SDKMAN/brew), PHP, Elixir, Kotlin, Swift, Dart/Flutter, R, and multi-language managers (mise, asdf).
- **Homebrew:** `brew leaves --installed-on-request` for user-installed formulae (plain `brew leaves` drops ones other packages depend on), `brew list --cask` for GUI apps, `brew tap`.
- **Package managers:** npm globals, pip globals, gem globals, cargo installs, go installs.
- **Databases:** PostgreSQL, MySQL, MongoDB, Redis, SQLite, MariaDB, CockroachDB, plus databases inside Docker volumes.
- **Cloud and DevOps CLIs:** AWS, GCP, Azure, Fly.io, Heroku, Vercel, Netlify, Stripe, Terraform, kubectl, helm, etc.
- **Mobile dev:** Xcode, Android Studio, CocoaPods, Flutter, fastlane.
- **Shell:** zsh, bash or fish, the framework (oh-my-zsh, prezto, starship), and what the config loads.

---

## Phase 2: Check Unpushed Git Work

Scan every project directory for repos with work that only exists on this Mac. Substitute the quoted paths from the state file:

```bash
find "/Users/OLD_USER/code" "/Users/OLD_USER/work" -maxdepth 5 -name .git -type d -prune 2>/dev/null | sed 's|/\.git$||' | while IFS= read -r repo; do
  unpushed=$(git -C "$repo" log --oneline --branches --not --remotes 2>/dev/null | wc -l | tr -d ' ')
  dirty=$(git -C "$repo" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
  no_upstream=$(git -C "$repo" for-each-ref --format='%(upstream)' refs/heads 2>/dev/null | grep -c '^$')
  stashes=$(git -C "$repo" stash list 2>/dev/null | wc -l | tr -d ' ')
  if [ "$unpushed" -gt 0 ] || [ "$dirty" -gt 0 ] || [ "$no_upstream" -gt 0 ] || [ "$stashes" -gt 0 ]; then
    echo "⚠️  $repo"
    [ "$unpushed" -gt 0 ] && echo "   unpushed commits: $unpushed"
    [ "$dirty" -gt 0 ] && echo "   uncommitted changes: $dirty files"
    [ "$no_upstream" -gt 0 ] && echo "   branches without upstream: $no_upstream"
    [ "$stashes" -gt 0 ] && echo "   stashes: $stashes"
  fi
done
```

1. Show the results. Ask the user to push, or to acknowledge, each repo before continuing.
2. For repos that can't be pushed (uncommitted WIP, stashes), a full rsync in Phase 4 preserves them exactly, including history, stashes and local branches.
3. Record the total repo count (`find ... | wc -l`) in the state file.

---

## Phase 3: Generate Setup Script

1. Read `references/setup-script-template.md`. Generate a personalized `~/new-mac-setup.sh` from the Phase 1 audit, including only tools that were actually detected.
2. Show the script and ask: "Does this look right? Any tools to add or remove?"
3. Save it to `~/new-mac-setup.sh`. Phase 4 transfers it, and the user runs it on the new Mac before Phase 5.

Generation rules:
- **Brew packages:** put formulae and casks in the zsh arrays; the template explains why.
- **PHP:** versions older than what brew supports natively come from the `shivammathur/php` tap.
- **MySQL 5.7 (EOL):** install `mysql@8.4` (LTS), not plain `mysql`, which is now 9.x and has dropped `mysql_native_password`. Warn the user about the upgrade.
- **Runtimes:** add `nvm install X` / `pyenv install X` / `rbenv install X` for each version detected.
- **Global packages:** reinstall them (npm -g, pip, gem, cargo).
- **Databases:** install the same or a newer major version than the old Mac (PostgreSQL restore requires this). Start every detected database service at the end.
- **Shell config:** install oh-my-zsh (or another framework) only if detected. **Do not write `.zshrc`** — Phase 7 copies the real one, and Phase 9 optimizes it.

---

## Phase 4: Transfer Files

Read `references/transfer.md` and follow it. Key points:
- The new Mac is the destination. You need **its** IP address, and Remote Login enabled **on the new Mac**.
- Set up a dedicated migration key and the `newmac` SSH alias. The user installs the public key by pasting one line in Terminal on the new Mac, so no password prompt is needed.
- Transfer the state file and setup script first. The user then runs the setup script on the new Mac (in parallel with the bigger transfers); it must finish before Phase 5.
- Before copying, check whether destination folders are synced by iCloud or OneDrive on the new Mac, and stop apps, dev servers and containers that write to files being copied.
- By default, exclude rebuildable, architecture-specific folders (`node_modules`, `.venv`, …). Show the exclude list and let the user adjust it.
- Large transfers run in the background with logs (see Long-running commands).

---

## Phase 5: Migrate Databases

Read `references/database-migration.md` and follow it. Key points:
- List the databases of each detected engine with sizes. Let the user pick which to migrate.
- MySQL: dump user databases only, never the `mysql` system schema.
- PostgreSQL: per-database `pg_dump -Fc` plus globals. If usernames differ, reassign ownership after the restore.
- Check every dump for errors before restoring. Errors go to log files, never `/dev/null`.
- Cover databases in Docker volumes too, and offer to copy tuned config settings.
- Record database counts per engine in the state file.

---

## Phase 6: Copy Apps and App Data

Read `references/apps.md` and follow it. Key points:
- Suggest a strategy per app (install fresh / copy app + data / data only / skip). The user confirms each one.
- Quit each app on both Macs before copying its data.
- Chrome cookies and saved passwords are encrypted with an old-Mac Keychain key and won't decrypt on the new Mac. Recommend Chrome Sync instead.
- `~/Library/Containers` and other protected folders need Full Disk Access for the terminal (old Mac) and for remote users (new Mac).

---

## Phase 7: Copy Dotfiles

Read `references/dotfiles.md` and follow it. Key points:
- **SSH:** never overwrite `authorized_keys`, never overwrite keys that already exist on the new Mac, and merge `known_hosts`. Don't copy the migration key.
- **Symlinked dotfiles (stow, chezmoi, yadm):** a copied symlink points nowhere on the new Mac. Detect them and re-link them.
- **Homebrew on PATH:** on an Intel → Apple Silicon move, make sure the new `~/.zprofile` loads `/opt/homebrew`.
- **Path fixes:** when usernames differ, run them on the new Mac over SSH, never locally. Find every copied file that mentions the old path, not just the shell config.
- **GPG:** pipe secret keys over SSH. Never leave an unencrypted export on disk.
- **Also cover:** global gitignore, oh-my-zsh custom plugins, crontab, user LaunchAgents, fonts, VS Code extensions, and custom `/etc/hosts` entries.

---

## Phase 8: Migrate Claude Code Data

Read `references/claude-data.md` and follow it. Key points:
- Copy `~/.claude/` (excluding `ide/` and `cache/`) **and** `~/.claude.json`.
- If usernames differ, only rewrite `/Users/OLD_USER` and the encoded project-folder form, never the bare username. In the encoded form, every non-alphanumeric character becomes `-`, so `/Users/john.doe` becomes `-Users-john-doe`.
- Finish by copying the final state file. Then tell the user to continue on the new Mac: run `claude`, then `/alb:mac-migration shell`.

---

## Phase 9: Optimize Shell (only if needed)

Runs on the new Mac. **Only suggest this phase if:**
- shell startup takes more than 0.5 seconds, OR
- heavy loaders appear in `.zshrc`: `nvm`, `pyenv`, `rbenv`, `rvm`, `jenv`, `sdkman`, `asdf`.

Read `references/shell-optimization.md` for the patterns.

1. Measure startup 3 times: `for i in 1 2 3; do /usr/bin/time zsh -i -c exit; done`
2. Identify slow loaders, using `zprof` if the cause isn't obvious.
3. Offer replacements: put shims or the default version on PATH so tools work in non-interactive shells (git hooks, editors), and lazy-load only the version manager command itself.
4. Back up `.zshrc`, then apply only the changes the user agrees to.
5. Measure again to confirm the improvement.

---

## Phase 10: Re-authenticate and Rebuild

Runs on the new Mac. Generate a checklist with only the CLIs detected in Phase 1:

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

Then the rebuild checklist for the folders excluded in Phase 4:
- **Node projects:** `npm install` / `yarn` / `pnpm install`. Native modules are rebuilt for the new architecture.
- **Python projects:** recreate virtualenvs (`python -m venv .venv && .venv/bin/pip install -r requirements.txt`, or `uv sync` / `poetry install`). Copied venvs point at old-user and old-arch interpreters.
- **iOS projects:** `pod install`. **Rust:** `cargo build`.
- **Docker:** re-pull or rebuild images. Images built on Intel run under emulation on Apple Silicon.

Also remind:
- Check `.env` files for machine-specific paths.
- Re-activate machine-locked licenses (JetBrains, Adobe, Sketch, etc.); deactivate them on the old Mac first.
- Test one project end-to-end.

---

## Phase 11: Verify Migration

Runs on the new Mac. Read `references/verify.md` and follow it. It compares the new Mac against the counts in the state file, produces a verification report, and finishes with cleanup: the migration SSH key, and the database dumps once the user confirms.
