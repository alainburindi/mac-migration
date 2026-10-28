# Audit Commands — Phase 1

Run these on the **old Mac** to audit the dev environment. All commands are read-only.

Depending on how the Bash tool's shell is set up, shell functions from `.zshrc` (`nvm`, `sdk`, lazy-load wrappers) may be missing, or may load slowly the first time. That's why version managers are detected by their directories as well as by their commands.

## System Info
```bash
echo "=== System ===" && sw_vers && uname -m && whoami
```

## Homebrew
```bash
echo "=== Homebrew Formulae ===" && brew leaves --installed-on-request 2>/dev/null
echo "=== Homebrew Casks ===" && brew list --cask 2>/dev/null
echo "=== Homebrew Taps ===" && brew tap 2>/dev/null
```

## Node
```bash
echo "=== Node ===" && node --version 2>/dev/null || echo "not found"
echo "=== NVM versions ===" && ls ~/.nvm/versions/node 2>/dev/null || echo "nvm not found"
echo "=== fnm versions ===" && fnm list 2>/dev/null || echo "fnm not found"
echo "=== Global npm packages ===" && npm list -g --depth=0 2>/dev/null
echo "=== Bun ===" && bun --version 2>/dev/null || echo "not found"
echo "=== Deno ===" && deno --version 2>/dev/null || echo "not found"
```

## Python
```bash
echo "=== Python ===" && python3 --version 2>/dev/null || echo "not found"
echo "=== pyenv versions ===" && pyenv versions 2>/dev/null || echo "pyenv not found"
echo "=== pip globals ===" && pip3 list 2>/dev/null | grep -v "^Package\|^---"
echo "=== Poetry ===" && poetry --version 2>/dev/null || echo "not found"
echo "=== uv ===" && uv --version 2>/dev/null || echo "not found"
```

## Ruby
```bash
echo "=== Ruby ===" && ruby --version 2>/dev/null || echo "not found"
echo "=== rbenv versions ===" && rbenv versions 2>/dev/null || echo "rbenv not found"
echo "=== rvm versions ===" && rvm list 2>/dev/null || echo "rvm not found"
echo "=== Global gems ===" && gem list --local 2>/dev/null | head -20
```

## PHP
```bash
echo "=== PHP ===" && php --version 2>/dev/null || echo "not found"
echo "=== PHP versions ===" && brew list | grep php 2>/dev/null
echo "=== Composer ===" && composer --version 2>/dev/null || echo "not found"
```

## Go
```bash
echo "=== Go ===" && go version 2>/dev/null || echo "not found"
echo "=== Go binaries ===" && ls ~/go/bin 2>/dev/null || echo "not found"
```

## Rust
```bash
echo "=== Rust ===" && rustc --version 2>/dev/null || echo "not found"
echo "=== Cargo installs ===" && ls ~/.cargo/bin 2>/dev/null || echo "not found"
```

## Java
```bash
echo "=== Java ===" && java --version 2>/dev/null || echo "not found"
echo "=== jenv versions ===" && jenv versions 2>/dev/null || echo "jenv not found"
echo "=== SDKMAN ===" && ls ~/.sdkman/candidates/java 2>/dev/null || echo "not found"
echo "=== Homebrew Java ===" && brew list | grep openjdk 2>/dev/null
```

## Multi-language Version Managers
```bash
echo "=== mise ===" && (mise ls 2>/dev/null || ls ~/.local/share/mise/installs 2>/dev/null) || echo "not found"
echo "=== asdf ===" && (asdf list 2>/dev/null || ls ~/.asdf/installs 2>/dev/null) || echo "not found"
echo "=== Volta ===" && (volta list all 2>/dev/null || ls ~/.volta/tools/image 2>/dev/null) || echo "not found"
echo "=== conda ===" && (conda env list 2>/dev/null || ls -d ~/miniconda3 ~/anaconda3 ~/miniforge3 2>/dev/null) || echo "not found"
```

## Other Languages
```bash
echo "=== Elixir ===" && elixir --version 2>/dev/null || echo "not found"
echo "=== Flutter/Dart ===" && flutter --version 2>/dev/null || echo "not found"
echo "=== Swift ===" && swift --version 2>/dev/null || echo "not found"
echo "=== Kotlin ===" && kotlinc -version 2>/dev/null || echo "not found"
echo "=== R ===" && R --version 2>/dev/null | head -1 || echo "not found"
```

## Databases
```bash
echo "=== PostgreSQL ===" && psql --version 2>/dev/null || echo "not found"
echo "=== MySQL ===" && mysql --version 2>/dev/null || echo "not found"
echo "=== MongoDB ===" && mongod --version 2>/dev/null || echo "not found"
echo "=== Redis ===" && redis-server --version 2>/dev/null || echo "not found"
echo "=== SQLite ===" && sqlite3 --version 2>/dev/null || echo "not found"
echo "=== Running services ===" && brew services list 2>/dev/null
echo "=== Docker volumes ===" && docker volume ls 2>/dev/null || echo "docker not running"
```

## Cloud & DevOps CLIs
```bash
echo "=== AWS ===" && aws --version 2>/dev/null || echo "not found"
echo "=== GCloud ===" && gcloud --version 2>/dev/null | head -1 || echo "not found"
echo "=== Azure ===" && az --version 2>/dev/null | head -1 || echo "not found"
echo "=== Fly.io ===" && flyctl version 2>/dev/null || echo "not found"
echo "=== Heroku ===" && heroku --version 2>/dev/null || echo "not found"
echo "=== Vercel ===" && vercel --version 2>/dev/null || echo "not found"
echo "=== Netlify ===" && netlify --version 2>/dev/null || echo "not found"
echo "=== Stripe ===" && stripe --version 2>/dev/null || echo "not found"
echo "=== Docker ===" && docker --version 2>/dev/null || echo "not found"
echo "=== kubectl ===" && kubectl version --client 2>/dev/null || echo "not found"
echo "=== Helm ===" && helm version 2>/dev/null || echo "not found"
echo "=== Terraform ===" && terraform --version 2>/dev/null || echo "not found"
echo "=== ngrok ===" && ngrok --version 2>/dev/null || echo "not found"
```

## Mobile Dev
```bash
echo "=== Xcode ===" && xcodebuild -version 2>/dev/null || echo "not found"
echo "=== CocoaPods ===" && pod --version 2>/dev/null || echo "not found"
echo "=== Fastlane ===" && fastlane --version 2>/dev/null || echo "not found"
echo "=== Android SDK ===" && echo $ANDROID_HOME
```

## Shell & Dotfiles

Lines that look like secrets (tokens, keys, passwords, URLs with embedded credentials) are filtered out so they never land in the conversation transcript.
```bash
echo "=== Shell ===" && echo $SHELL
echo "=== Framework ===" && ls -d ~/.oh-my-zsh ~/.zprezto ~/.config/starship.toml 2>/dev/null
for f in ~/.zshrc ~/.zprofile ~/.zshenv ~/.gitconfig; do
  [ -f "$f" ] && echo "=== $f (secrets filtered) ===" && /usr/bin/grep -viE 'token|secret|passw|api_?key|_key *=|auth|credential|bearer|_pat\b|_dsn|://[^ /]*@' "$f"
done
echo "=== SSH keys ===" && ls ~/.ssh/ 2>/dev/null
echo "=== GPG secret keys ===" && gpg --list-secret-keys --keyid-format long 2>/dev/null || echo "gpg not found"
```

## Scheduled Jobs & Other User Config
```bash
echo "=== crontab ===" && crontab -l 2>/dev/null || echo "none"
echo "=== LaunchAgents ===" && ls ~/Library/LaunchAgents 2>/dev/null
echo "=== Custom /etc/hosts entries ===" && /usr/bin/grep -vE '^\s*(#|$)|localhost|broadcasthost' /etc/hosts
echo "=== User fonts ===" && ls ~/Library/Fonts 2>/dev/null | wc -l
echo "=== VS Code extensions ===" && code --list-extensions 2>/dev/null | wc -l
echo "=== ~/.config ===" && ls ~/.config 2>/dev/null
```
