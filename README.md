# /alb:mac-migration

An interactive Claude Code skill that guides you through migrating your complete developer environment from an old Mac to a new Mac — **without using Migration Assistant**.

## Installation

**Step 1** — Add this repo as a marketplace (one-time):
```
/plugin marketplace add alainburindi/mac-migration
```

**Step 2** — Install the plugin:
```
/plugin install alb@mac-migration
```

**Step 3** — Run it on the **old** Mac:
```
/alb:mac-migration
```

Or jump directly to a specific phase (progress is kept in `~/mac-migration-state.md`, so you can resume in a new session):
```
/alb:mac-migration audit            # Phase 1: Audit old Mac
/alb:mac-migration git-check        # Phase 2: Check unpushed git work
/alb:mac-migration generate-script  # Phase 3: Generate setup script
/alb:mac-migration transfer         # Phase 4: Transfer files
/alb:mac-migration databases        # Phase 5: Migrate databases
/alb:mac-migration apps             # Phase 6: Copy apps and data
/alb:mac-migration dotfiles         # Phase 7: Copy dotfiles
/alb:mac-migration claude-data      # Phase 8: Migrate Claude Code data
/alb:mac-migration shell            # Phase 9: Optimize shell (new Mac)
/alb:mac-migration auth             # Phase 10: Re-authenticate (new Mac)
/alb:mac-migration verify           # Phase 11: Verify migration (new Mac)
```

## When to use this

Migration Assistant is great when both Macs have the same username and you want to copy everything. But it doesn't work well when:

- **Your usernames are different** — e.g. personal Mac (`john`) → work Mac (`jsmith`)
- **It's a corporate/work machine** — IT assigned your username, you can't change it
- **You want full control** — choose exactly what gets migrated, skip what you don't need
- **You're doing a clean install** — start fresh but bring over your tools and data selectively

This skill handles all of these scenarios, including automatically fixing hardcoded paths when usernames differ.

## What it migrates

| Category | What |
|---|---|
| Dev tools | Homebrew packages, any language runtime (Node, Python, Ruby, Go, Rust, Java, PHP, etc.) |
| Projects | All git repos, preserving local branches and uncommitted work |
| Databases | PostgreSQL, MySQL, MongoDB, Redis, SQLite, Docker volumes — any engine |
| Apps | Suggests copy vs reinstall for each app, copies app data (sessions, connections, history) |
| Dotfiles | `.zshrc`, `.gitconfig`, SSH & GPG keys, shell history (cleaned), crontab, LaunchAgents, fonts |
| Claude Code | Sessions, memory, project history, plugins — with path fixes |
| Shell | Optionally optimizes shell startup time (lazy loading) |

## The 11 Phases

| Phase | What happens |
|---|---|
| **1. Audit** | Scans your old Mac — detects every language runtime, tool, database, and CLI installed |
| **2. Git check** | Scans your project directories for unpushed commits, local-only branches, and uncommitted changes |
| **3. Setup script** | Generates a personalized `new-mac-setup.sh` based on exactly what you have |
| **4. Transfer** | Transfers files via rsync over WiFi or Thunderbolt cable, skipping rebuildable folders like `node_modules` |
| **5. Databases** | Dumps and restores all databases, handles version compatibility (e.g. MySQL 5.7 → 8) |
| **6. Apps** | Analyzes each app and suggests copy vs reinstall, copies all app data |
| **7. Dotfiles** | Copies SSH keys, git config, shell config and history (with optional cleanup) |
| **8. Claude Code** | Copies sessions, memory, projects — fixes hardcoded paths if username changed |
| **9. Shell** | Measures startup time and applies lazy-loading optimizations (only if needed) |
| **10. Re-auth** | Generates a checklist of CLIs that need re-authentication and dependencies to rebuild |
| **11. Verify** | Runs checks on the new Mac, produces a verification report, and cleans up |

Phases 1–8 run from the old Mac. Phases 9–11 run in Claude Code on the new Mac.

## Requirements

- Both Macs on the same WiFi network, OR connected via a Thunderbolt cable
- Remote Login enabled on the new Mac (`System Settings > General > Sharing > Remote Login`), with "Allow full disk access for remote users" turned on
- Full Disk Access for your terminal app on the old Mac
- Claude Code installed on the old Mac (the setup script can install it on the new Mac)

## Known limitations

- App Store apps cannot be copied — they need to be re-downloaded from the App Store
- Apps with machine-locked licenses (Adobe, some JetBrains products) need to be deactivated on the old Mac first
- MySQL 5.7 is EOL and not available on newer Macs — the skill will upgrade you to MySQL 8
- PHP versions older than 7.4 require the `shivammathur/php` brew tap
- Passwords, cookies and tokens stored in the macOS Keychain (Chrome sessions, DB GUI passwords, CLI credentials) don't transfer — expect to sign in again
- Commands that need a password (`sudo`, the setup script) are run by you in Terminal; the skill tells you when

## Real-world example

This skill was built from a real migration:
- Old Mac: MacBook Pro, username `alainburindi`
- New Mac: M5, username `alabu` (work-assigned)
- ~39GB of projects across 40+ git repos
- PostgreSQL (29 databases), MongoDB, MySQL
- 50+ apps

## License

MIT
