---
title: "Managing Development Language Versions with asdf"
description: "Install and configure asdf on macOS ARM64 to manage Ruby, Python, Node.js, Golang, Java, and other runtime versions with one tool"
date: 2026-04-25T10:00:00+08:00
slug: asdf-version-manager
image: "cover.svg"
categories:
    - Tech
tags:
    - macOS
    - asdf
    - Ruby
    - Python
    - Node.js
    - Golang
    - Java
---

In [a previous post](/p/macos-dev-env-setup/), I used rbenv, pyenv, and nvm separately to manage Ruby, Python, and Node.js versions. That works, but each tool has its own installation flow, configuration style, and command set, which adds overhead as the number of languages grows.

[asdf](https://asdf-vm.com/) is an extensible version manager that provides a unified command interface and plugin system, allowing one tool to manage runtime versions for many languages and CLI tools. This post walks through installing asdf on macOS arm64 and configuring it for Ruby, Python, Node.js, Golang, and Java.

<!--more-->

## Why use asdf

Compared with installing a separate version manager for each language, asdf has several practical advantages:

- A unified command model, with consistent commands such as `asdf install`, `asdf set`, and `asdf current`
- A single configuration file, where one `.tool-versions` file can declare multiple runtimes
- A wide plugin ecosystem that supports not only programming languages but many common developer tools
- A much better fit for polyglot repositories that depend on Node.js, Ruby, Java, and Go at the same time
- Easier team onboarding, since committing `.tool-versions` lets others run `asdf install` and align their environment quickly

It is not perfect. asdf is a general framework, and much of the language-specific behavior comes from plugins, so plugin quality and maintenance vary. Even so, for engineers who regularly work across multiple runtimes, asdf is often simpler to maintain than juggling several dedicated version managers.

## 1. Prerequisites

### 1.1 Install Xcode Command Line Tools

Many runtime versions on Apple Silicon need to be built from source, so install the command-line developer tools first:

```bash
xcode-select --install
```

If they are already installed, macOS will tell you and you can move on.

### 1.2 Install Homebrew

This post assumes Homebrew is already installed. If not, see my earlier guide on [setting up a macOS ARM64 development environment](/p/macos-dev-env-setup/).

Verify Homebrew first:

```bash
brew --version
```

### 1.3 Install common build dependencies

Ruby, Python, Java, and other runtimes often depend on system libraries during installation, so it helps to install a common baseline up front:

```bash
brew install coreutils curl git openssl readline sqlite3 xz zlib tcl-tk libyaml gmp autoconf automake libtool
```

Not every runtime needs all of these packages, but having them available reduces the chance of build failures later.

## 2. Install asdf

### 2.1 Install via Homebrew

On macOS, the most direct approach is to install asdf with Homebrew:

```bash
brew install asdf
```

Then verify it:

```bash
asdf --version
```

### 2.2 Configure your shell

If you use zsh, add the following to `~/.zshrc`:

```bash
# asdf
export ASDF_DATA_DIR="$HOME/.asdf"
export PATH="$ASDF_DATA_DIR/shims:$PATH"
```

Then reload your shell configuration:

```bash
source ~/.zshrc
```

Two details matter here:

- `ASDF_DATA_DIR` defines where asdf stores its plugins, installed versions, and shims
- `shims` must be in `PATH`, otherwise commands like `ruby`, `python`, `node`, `go`, and `java` will not be routed through asdf

Some installation methods still mention `asdf init`, but with current asdf and Homebrew setups, keeping the executable and shims in `PATH` is typically enough.

### 2.3 Verify the installation

```bash
which asdf
asdf info
```

`asdf info` is especially useful when troubleshooting, because it prints the OS, shell, and asdf-related paths.

## 3. How asdf works

Before installing language runtimes, it helps to understand the basic model.

### 3.1 Plugins

asdf itself handles version resolution and shims. Installing a specific language depends on a plugin.

Examples:

- `ruby` for Ruby
- `python` for Python
- `nodejs` for Node.js
- `golang` for Go
- `java` for Java

### 3.2 Version scope

asdf usually works across three scopes:

- Global version, used for your default shell environment
- Local version, used only in the current directory and its children
- Temporary shell version, used only in the current shell session

### 3.3 `.tool-versions`

The most important asdf configuration file is `.tool-versions`. For example:

```text
ruby 3.3.7
python 3.13.3
nodejs 22.14.0
golang 1.24.2
java temurin-21.0.7+6.0.LTS
```

This file commonly lives in:

- Your home directory, for user-wide defaults
- A project root, for project-specific runtime pinning

When you enter a directory, asdf searches upward for `.tool-versions` and determines which runtime versions should be active.

### 3.4 Common commands

```bash
# Add a plugin
asdf plugin add <name>

# List installed plugins
asdf plugin list

# List available versions
asdf list all <name>

# Install a specific version
asdf install <name> <version>

# Set a version for the current directory
asdf set <name> <version>

# Set a user-wide default version
asdf set -u <name> <version>

# Show active versions
asdf current

# List installed versions for one tool
asdf list <name>
```

In practice:

- `asdf set <name> <version>` writes to `.tool-versions` in the current directory
- `asdf set -u <name> <version>` writes to the user-level config

## 4. Install and manage Ruby

### 4.1 Add the Ruby plugin

```bash
asdf plugin add ruby
```

Confirm that it is installed:

```bash
asdf plugin list
```

### 4.2 Install Ruby build dependencies

On macOS ARM64, Ruby builds commonly depend on OpenSSL, libyaml, readline, and related libraries:

```bash
brew install openssl readline libyaml gmp
```

### 4.3 Install a Ruby version

```bash
# List available versions
asdf list all ruby

# Install one version
asdf install ruby 3.3.7
```

If installation feels slow, that is normal. Ruby often builds from source.

### 4.4 Set the version and verify it

Set a global default:

```bash
asdf set -u ruby 3.3.7
```

Set a project version:

```bash
asdf set ruby 3.3.7
```

Verify:

```bash
ruby --version
which ruby
asdf current ruby
```

### 4.5 Bundler and gem usage

After Ruby is installed, it is a good idea to update RubyGems and Bundler:

```bash
gem update --system
gem install bundler
```

Under asdf, `gem install` typically writes into the active Ruby version's own directory, so `sudo` is usually unnecessary.

## 5. Install and manage Python

### 5.1 Add the Python plugin

```bash
asdf plugin add python
```

### 5.2 Install Python build dependencies

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

### 5.3 Install Python

```bash
# List available versions
asdf list all python

# Install one version
asdf install python 3.13.3
```

### 5.4 Set the version and verify it

```bash
asdf set -u python 3.13.3
python --version
pip --version
which python
asdf current python
```

For a project-level Python version, run this inside the project directory:

```bash
asdf set python 3.13.3
```

### 5.5 Virtual environment guidance

asdf manages the Python interpreter version, not your project dependency isolation. In real projects, you should still pair it with tools such as:

- `python -m venv`
- `poetry`
- `uv`
- `pipenv`

In other words:

- asdf decides which Python interpreter you use
- a virtual environment tool decides where the project dependencies live

Those are separate layers and should stay separate.

## 6. Install and manage Node.js

### 6.1 Add the Node.js plugin

```bash
asdf plugin add nodejs
```

### 6.2 Import the Node.js release team keys

The Node.js plugin usually requires importing the release team's GPG keys so downloads can be verified:

```bash
bash ~/.asdf/plugins/nodejs/bin/import-release-team-keyring
```

If this is your first Node.js installation through asdf, do not skip this step.

### 6.3 Install Node.js

```bash
# List available versions
asdf list all nodejs

# Install one version
asdf install nodejs 22.14.0
```

### 6.4 Set the version and verify it

```bash
asdf set -u nodejs 22.14.0
node --version
npm --version
which node
asdf current nodejs
```

If a project already uses `.nvmrc`, you can gradually migrate by copying that version into `.tool-versions`.

### 6.5 Corepack recommendation

If your project uses `pnpm` or `yarn`, enable Corepack:

```bash
corepack enable
```

That keeps the package manager version aligned more cleanly with the active Node.js runtime.

## 7. Install and manage Golang

### 7.1 Add the Go plugin

```bash
asdf plugin add golang
```

### 7.2 Install Go

```bash
# List available versions
asdf list all golang

# Install one version
asdf install golang 1.24.2
```

### 7.3 Set the version and verify it

```bash
asdf set -u golang 1.24.2
go version
which go
asdf current golang
```

### 7.4 GOPATH and GOBIN guidance

Modern Go projects usually do not need heavy manual `GOPATH` configuration, but for compatibility with some tools you may still want this in `~/.zshrc`:

```bash
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"
```

That makes commands installed with `go install` available in your shell.

## 8. Install and manage Java

### 8.1 Add the Java plugin

```bash
asdf plugin add java
```

### 8.2 List Java distributions and versions

The Java plugin supports several distributions, including:

- temurin
- corretto
- zulu
- openjdk

List available versions first:

```bash
asdf list all java
```

The output is often long, so filtering helps:

```bash
asdf list all java | grep temurin
```

### 8.3 Install Java

For example, install Temurin 21:

```bash
asdf install java temurin-21.0.7+6.0.LTS
```

### 8.4 Set the version and verify it

```bash
asdf set -u java temurin-21.0.7+6.0.LTS
java -version
which java
asdf current java
```

### 8.5 `JAVA_HOME` guidance

Many Java tools still expect `JAVA_HOME`. One possible `~/.zshrc` approach is:

```bash
export JAVA_HOME="$HOME/.asdf/installs/java/$(asdf current java | awk '{print $2}')"
```

That said, this depends on `asdf` already working during shell startup. A more predictable approach is to:

- pin the Java version in `.tool-versions`
- configure `JAVA_HOME` per project, per IDE, or in build tooling where needed

For IDEs such as IntelliJ IDEA, selecting the JDK directly from the asdf install directory is often the clearest setup.

## 9. Example `.tool-versions` for a polyglot project

Suppose a repository contains:

- a Rails backend
- Python data scripts
- a Node.js frontend build
- Go-based tooling
- a Java service

Then the project root `.tool-versions` might look like this:

```text
ruby 3.3.7
python 3.13.3
nodejs 22.14.0
golang 1.24.2
java temurin-21.0.7+6.0.LTS
```

Then run:

```bash
asdf install
```

asdf will install any missing versions declared in `.tool-versions`. This is one of the strongest parts of the tool: environment declaration and environment setup are tightly connected.

## 10. Troubleshooting

### 10.1 The runtime version is not what you expect

Check what asdf thinks is active:

```bash
asdf current
```

Then check the actual binary path:

```bash
which ruby
which python
which node
which go
which java
```

If those paths do not point into `~/.asdf/shims/...`, the issue is usually your `PATH` order.

### 10.2 The plugin is installed but the command is missing

Try rebuilding the shims:

```bash
asdf reshim
```

This is especially useful after installing Ruby gems or global Node.js commands that expose executables.

### 10.3 Installation fails

The three most common causes are:

- Xcode Command Line Tools are missing or broken
- Homebrew dependencies are missing
- An older runtime version has poor Apple Silicon compatibility

Start with:

```bash
asdf info
brew doctor
xcode-select -p
```

### 10.4 Older versions fail on ARM64

This is common on Apple Silicon, especially with old Ruby, Python, or Node.js releases. A practical troubleshooting order is:

- prefer newer maintained versions first
- check issues in the relevant plugin repository
- add build flags for specific versions when required
- use a Rosetta-based x86_64 environment only as a last resort

In general, it is better not to run your whole terminal under Rosetta just to support a small number of legacy tools.

## 11. Migrating from rbenv, pyenv, or nvm

If your machine already uses rbenv, pyenv, or nvm, do not remove everything at once. A safer migration path looks like this:

1. Install asdf and the plugins you need
2. Install runtime versions matching your current projects
3. Create `.tool-versions` in the relevant repositories
4. Open a fresh shell and confirm that `ruby`, `python`, `node`, and other commands are now coming from asdf
5. Remove old version-manager initialization from your shell only after everything works

The most common migration bug is `PATH` ordering. When multiple version managers all try to modify `PATH`, it is easy to install something successfully and still not be using it.

## 12. Recommended `~/.zshrc` example

If your setup mainly depends on Homebrew and asdf, a compact example looks like this:

```bash
# Homebrew
export HOMEBREW_PREFIX="/opt/homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"

# asdf
export ASDF_DATA_DIR="$HOME/.asdf"
export PATH="$ASDF_DATA_DIR/shims:$PATH"

# Go tools
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"
```

If your Homebrew lives in a user directory, replace `HOMEBREW_PREFIX` with your actual installation path.

## 13. Summary

The real value of asdf on macOS ARM64 is not just that it supports many languages. It gives version management a consistent workflow: add a plugin, install a version, write `.tool-versions`, and let directory-aware switching do the rest.

When a project depends on Ruby, Python, Node.js, Go, and Java together, that consistency noticeably reduces environment maintenance cost. For team onboarding and setting up a new machine, asdf is often easier to maintain over time than a stack of separate version managers.
