# Shell Optimization Reference — Phase 9

Runs on the new Mac. Back up first: `cp ~/.zshrc ~/.zshrc.pre-optimize`.

## Measuring startup time

```bash
for i in 1 2 3; do /usr/bin/time zsh -i -c exit; done
```
Anything over 0.5s is worth optimizing.

## Principle: fast PATH, lazy manager

Don't wrap `node`, `python`, `ruby` or `java` in lazy-load shell functions. Shell functions only exist in interactive shells, so git hooks (husky, pre-commit), editors, `#!/usr/bin/env node` scripts and other non-interactive tools would not find the runtime at all.

Instead:
1. Put the default version (or the manager's shims directory) on `PATH`. This costs almost nothing.
2. Lazy-load only the version manager command itself (`nvm`, `pyenv`, …), which is the slow part and only needed when switching versions.

---

## NVM (typically adds 300ms–1s)

**Before (slow):**
```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

**After:**
```zsh
export NVM_DIR="$HOME/.nvm"
# Default Node on PATH: the version matching `nvm alias default`, else the highest installed
_nvm_default=$(cat "$NVM_DIR/alias/default" 2>/dev/null)
_nvm_bins=("$NVM_DIR"/versions/node/v${_nvm_default#v}*(N/nOn) "$NVM_DIR"/versions/node/v*(N/nOn))
(( ${#_nvm_bins} )) && export PATH="${_nvm_bins[1]}/bin:$PATH"
unset _nvm_default _nvm_bins
# Load nvm itself only when it's used
nvm() { unset -f nvm; [ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"; nvm "$@"; }
```
This is zsh syntax: `(N/nOn)` = no error if no match, directories only, numeric sort descending.

---

## pyenv (typically adds 200–500ms)

**Before (slow):**
```bash
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

**After:**
```zsh
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/shims:$PYENV_ROOT/bin:$PATH"
pyenv() {
  unset -f pyenv
  eval "$(command pyenv init -)"
  command pyenv commands | grep -qx virtualenv-init && eval "$(command pyenv virtualenv-init -)"
  pyenv "$@"
}
```
Shims on PATH make `python`, `pip` and any `.python-version` file work everywhere. Only `pyenv shell` and auto-activation of virtualenvs need the full init, which happens on the first `pyenv` call.

---

## rbenv (typically adds 100–300ms)

**Before (slow):**
```bash
eval "$(rbenv init -)"
```

**After:**
```zsh
export PATH="$HOME/.rbenv/shims:$PATH"
rbenv() { unset -f rbenv; eval "$(command rbenv init -)"; rbenv "$@"; }
```

---

## jenv (typically adds 100–200ms)

**Before (slow):**
```bash
eval "$(jenv init -)"
```

**After:**
```zsh
export PATH="$HOME/.jenv/shims:$PATH"
jenv() { unset -f jenv; eval "$(command jenv init -)"; jenv "$@"; }
```
If tools rely on `JAVA_HOME`, run `jenv enable-plugin export` once; it needs the full init, so keep the eager `eval` for that case.

---

## SDKMAN (typically adds 500ms+)

**Before (slow):**
```bash
export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$HOME/.sdkman/bin/sdkman-init.sh" ]] && source "$HOME/.sdkman/bin/sdkman-init.sh"
```

**After:**
```zsh
export SDKMAN_DIR="$HOME/.sdkman"
# Current version of every installed candidate (java, gradle, kotlin, …) on PATH
for _c in "$SDKMAN_DIR"/candidates/*/current(N); do export PATH="$_c/bin:$PATH"; done
unset _c
[ -d "$SDKMAN_DIR/candidates/java/current" ] && export JAVA_HOME="$SDKMAN_DIR/candidates/java/current"
sdk() { unset -f sdk; source "$SDKMAN_DIR/bin/sdkman-init.sh"; sdk "$@"; }
```

---

## Other optimizations

### Remove duplicate PATH entries
```zsh
typeset -U PATH path
```

### Defer non-critical plugins with `zsh-defer`
```zsh
# Install: git clone https://github.com/romkatv/zsh-defer.git ~/.zsh-defer
source ~/.zsh-defer/zsh-defer.plugin.zsh
zsh-defer source /path/to/slow/plugin.zsh
```

### Profile what's slow
```zsh
# Add to the top of .zshrc temporarily:
zmodload zsh/zprof

# Add to the bottom of .zshrc:
zprof
```
This prints a profile of every function called during startup, which pinpoints the exact culprit. Remove both lines afterwards.
