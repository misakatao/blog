---
title: "macOS ARM64 Dev Environment Setup: Homebrew, Ruby, Python, Node.js"
description: "Set up a complete development environment on macOS Apple Silicon with Homebrew in the user directory, plus rbenv, pyenv, and nvm for multi-version runtime management"
date: 2026-04-22T09:00:00+08:00
slug: macos-dev-env-setup
image: "cover.svg"
categories:
    - Tech
tags:
    - macOS
    - Homebrew
    - Ruby
    - Python
    - Node.js
---

This post walks through setting up a development environment on macOS arm64 (Apple Silicon), installing Homebrew under the user's home directory, and using rbenv, pyenv, and nvm to manage Ruby, Python, and Node.js versions.

<!--more-->

## Why Install to the User Directory

By default, Homebrew installs to `/opt/homebrew` (Apple Silicon) or `/usr/local` (Intel), both requiring admin privileges. Installing to a user directory like `~/.homebrew` offers:

- No `sudo` required — everything stays in user space
- No interference with system-level package management
- Clean isolation in multi-user environments
- Easy to migrate or remove entirely

## 1. Installing Homebrew

### 1.1 Clone Homebrew to User Directory

```bash
git clone https://github.com/Homebrew/brew.git ~/.homebrew
```

### 1.2 Configure Environment Variables

Add to `~/.zshrc` (or `~/.bash_profile` for bash):

```bash
# Homebrew
export HOMEBREW_PREFIX="$HOME/.homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"
export HOMEBREW_CELLAR="$HOMEBREW_PREFIX/Cellar"
export HOMEBREW_REPOSITORY="$HOMEBREW_PREFIX"
export MANPATH="$HOMEBREW_PREFIX/share/man:$MANPATH"
export INFOPATH="$HOMEBREW_PREFIX/share/info:$INFOPATH"
```

| Variable | Purpose |
|----------|---------|
| `HOMEBREW_PREFIX` | Homebrew installation root |
| `PATH` | Prioritize Homebrew binaries over system ones |
| `HOMEBREW_CELLAR` | Where installed packages are stored |
| `HOMEBREW_REPOSITORY` | Location of the Homebrew core repo |
| `MANPATH` / `INFOPATH` | Documentation search paths |

Apply changes:

```bash
source ~/.zshrc
```

### 1.3 Verify

```bash
brew --version
brew doctor
```

## 2. Ruby with rbenv

### 2.1 Install rbenv and ruby-build

```bash
brew install rbenv ruby-build
```

### 2.2 Configure rbenv

Add to `~/.zshrc`:

```bash
# rbenv
eval "$(rbenv init - zsh)"
```

`rbenv init` prepends `~/.rbenv/shims` to PATH, installs shell functions for rehashing, and enables autocompletion.

```bash
source ~/.zshrc
```

### 2.3 Install Ruby

```bash
rbenv install -l          # list available versions
rbenv install 3.3.7       # install a specific version
rbenv global 3.3.7        # set as default
ruby --version            # verify
```

Use `rbenv local 3.x.x` in a project directory to pin a version via `.ruby-version`.

## 3. Python with pyenv

### 3.1 Install pyenv

```bash
brew install pyenv
```

### 3.2 Configure pyenv

Add to `~/.zshrc`:

```bash
# pyenv
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"
```

| Config | Purpose |
|--------|---------|
| `PYENV_ROOT` | Root directory where all Python versions are stored |
| `PATH` append | Ensures the `pyenv` binary is found |
| `pyenv init` | Adds shims to PATH for version switching |

### 3.3 Install Build Dependencies

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

### 3.4 Install Python

```bash
pyenv install -l | grep -E "^\s+3\."   # list Python 3.x versions
pyenv install 3.13.3
pyenv global 3.13.3
python --version
```

## 4. Node.js with nvm

Unlike rbenv/pyenv, nvm is not installed via Homebrew (officially discouraged). Use the install script instead.

### 4.1 Install nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
```

### 4.2 Verify Configuration

The script should have added this to `~/.zshrc`:

```bash
# nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

nvm is a pure shell function, not a binary — it's loaded into each shell session via `nvm.sh`.

### 4.3 Install Node.js

```bash
nvm ls-remote --lts       # list LTS versions
nvm install --lts         # install latest LTS
nvm alias default 22      # set default version
node --version
npm --version
```

## 5. Complete ~/.zshrc Summary

```bash
# Homebrew (user directory)
export HOMEBREW_PREFIX="$HOME/.homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"
export HOMEBREW_CELLAR="$HOMEBREW_PREFIX/Cellar"
export HOMEBREW_REPOSITORY="$HOMEBREW_PREFIX"
export MANPATH="$HOMEBREW_PREFIX/share/man:$MANPATH"
export INFOPATH="$HOMEBREW_PREFIX/share/info:$INFOPATH"

# rbenv
eval "$(rbenv init - zsh)"

# pyenv
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"

# nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

## 6. Notes

**Architecture**: Some older Ruby/Python/Node.js versions lack arm64 prebuilt binaries and will compile from source (slower). Ensure Xcode Command Line Tools are installed: `xcode-select --install`.

**User-directory Homebrew**: Not all bottles (prebuilt packages) are available for non-standard paths. Homebrew will fall back to source compilation when needed. Run `brew doctor` to check for issues.

**Version pinning**: Use `local` commands (`rbenv local`, `pyenv local`) and commit `.ruby-version`, `.python-version`, `.nvmrc` to version control for team consistency.

**No sudo**: Never use `sudo` with `gem install`, `pip install`, or `npm install -g` — the version managers already point these to user-space directories.

**Load order in ~/.zshrc**: Homebrew PATH first (rbenv/pyenv depend on it), then rbenv/pyenv init (they prepend shims), then nvm (independent).

**Troubleshooting**:

```bash
which ruby && ruby --version
which python && python --version
which node && node --version
rbenv versions
pyenv versions
nvm ls
echo $PATH | tr ':' '\n'
```
