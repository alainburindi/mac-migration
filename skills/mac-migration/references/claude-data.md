# Migrate Claude Code Data — Phase 8

Claude Code keeps its data in two places: the `~/.claude/` directory (sessions, memory, projects, plugins, settings) and the `~/.claude.json` file (MCP servers, per-project settings, account info). Credentials live in the macOS Keychain and do not transfer, so the user logs in again.

## 1. Check both sides

```bash
ls -la ~/.claude/ ~/.claude.json
ssh newmac 'ls -la ~/.claude/ ~/.claude.json 2>/dev/null'
```
If Claude Code has already been used on the new Mac, warn that this copy replaces its settings and merges session folders. Ask before continuing.

## 2. Copy

Exclude `ide/` (lock files for running editors) and `cache/`:
```bash
tar czf - -C ~ --exclude=".claude/ide" --exclude=".claude/cache" .claude | \
  ssh newmac 'tar xzf - -C ~'
rsync -avh ~/.claude.json newmac:
```

## 3. Fix hardcoded paths and rename project directories (only if usernames differ)

Claude Code stores absolute paths in two forms:
- `/Users/OLD_USER/...` in file contents
- an encoded form in project directory names, and in references to them. Every non-alphanumeric character becomes `-`, so `/Users/john.doe/code` becomes `-Users-john-doe-code`

Only rewrite those two forms, never the bare username, which can appear anywhere in session transcripts. Binary files are skipped.
```bash
ssh newmac 'python3 -' <<'EOF'
import os, re
OLD, NEW = 'OLD_USER', 'NEW_USER'
home = os.path.expanduser('~')
root = os.path.join(home, '.claude')
# Claude Code's project-folder encoding: every non-alphanumeric character becomes '-'
enc = lambda u: re.sub(r'[^A-Za-z0-9]', '-', u)
# Match only when the username is not followed by more username characters
path_pat = re.compile(r'/Users/' + re.escape(OLD) + r'(?![A-Za-z0-9_.-])')
enc_pat = re.compile(r'-Users-' + re.escape(enc(OLD)) + r'(?![A-Za-z0-9])')
fix = lambda s: enc_pat.sub(lambda m: '-Users-' + enc(NEW), path_pat.sub(lambda m: '/Users/' + NEW, s))

def rewrite(path):
    try:
        with open(path, encoding='utf-8', newline='') as f:
            text = f.read()
    except (UnicodeDecodeError, OSError):
        return False
    new_text = fix(text)
    if new_text == text:
        return False
    with open(path, 'w', encoding='utf-8', newline='') as f:
        f.write(new_text)
    return True

changed = sum(rewrite(p) for p in [os.path.join(home, '.claude.json')])
for dirpath, dirnames, filenames in os.walk(root):
    dirnames[:] = [d for d in dirnames if d not in ('cache', 'ide')]
    changed += sum(rewrite(os.path.join(dirpath, fn)) for fn in filenames)
print(f'Rewrote paths in {changed} files')

projects = os.path.join(root, 'projects')
for name in os.listdir(projects) if os.path.isdir(projects) else []:
    new_name = fix(name)
    if new_name == name:
        continue
    src, dst = os.path.join(projects, name), os.path.join(projects, new_name)
    if os.path.exists(dst):
        print(f'SKIPPED (already exists on new Mac): {name} -> {new_name}')
    else:
        os.rename(src, dst)
        print(f'Renamed: {name} -> {new_name}')
EOF
```
If any directory was skipped, show it to the user and ask whether to merge its contents into the existing one.

## 4. Verify

```bash
ssh newmac "/usr/bin/grep -rlI -e '/Users/OLD_USER/' -e '-Users-OLD_USER_ENCODED-' ~/.claude/ ~/.claude.json 2>/dev/null | wc -l"
# Should return 0. OLD_USER_ENCODED = OLD_USER with every non-alphanumeric character replaced by '-'.
```

## 5. Hand off to the new Mac

1. Update the state file (mark Phase 8 done, record the Claude project count from `ls ~/.claude/projects | wc -l`), then copy it over:
   ```bash
   rsync -avh ~/mac-migration-state.md newmac:
   ```
2. Tell the user to continue on the new Mac:
   - Install Claude Code there if the setup script didn't.
   - Run `claude` and log in (`/login`).
   - Run `/alb:mac-migration shell`. The plugin came over with `~/.claude/`, and the new session picks up the state file.
3. Tell the user that Phase 11 removes the migration SSH key and the `newmac` alias. If they stop before Phase 11, they should run its cleanup section anyway, because the key keeps granting the old Mac access to the new one.
