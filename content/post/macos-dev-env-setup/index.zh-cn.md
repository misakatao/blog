---
title: "macOS ARM64 开发环境搭建指南"
description: "在 macOS Apple Silicon 上从零搭建开发环境，将 Homebrew 安装到用户目录，并通过 rbenv、pyenv、nvm 管理多版本运行时"
date: 2026-04-22
slug: macos-dev-env-setup
image: "cover.svg"
categories:
    - 技术
tags:
    - macOS
    - Homebrew
    - Ruby
    - Python
    - Node.js
---

本文记录在 macOS arm64（Apple Silicon）系统上，将 Homebrew 安装到当前用户目录，并基于 Homebrew 安装 rbenv、pyenv、nvm 来管理 Ruby、Python、Node.js 多版本环境的完整过程。

<!--more-->

## 为什么安装到用户目录

Homebrew 官方默认将自身安装到 `/opt/homebrew`（Apple Silicon）或 `/usr/local`（Intel），这些路径需要管理员权限。将 Homebrew 安装到用户主目录下（如 `~/.homebrew`）有以下好处：

- 不需要 `sudo` 权限，所有操作在当前用户空间完成
- 不会影响系统级别的包管理
- 多用户环境下互不干扰
- 方便整体迁移或清理

## 一、安装 Homebrew

### 1.1 克隆 Homebrew 到用户目录

```bash
git clone https://github.com/Homebrew/brew.git ~/.homebrew
```

这会将 Homebrew 的核心仓库克隆到 `~/.homebrew` 目录。

### 1.2 配置环境变量

编辑 shell 配置文件（zsh 用户编辑 `~/.zshrc`，bash 用户编辑 `~/.bash_profile`）：

```bash
# Homebrew
export HOMEBREW_PREFIX="$HOME/.homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"
export HOMEBREW_CELLAR="$HOMEBREW_PREFIX/Cellar"
export HOMEBREW_REPOSITORY="$HOMEBREW_PREFIX"
export MANPATH="$HOMEBREW_PREFIX/share/man:$MANPATH"
export INFOPATH="$HOMEBREW_PREFIX/share/info:$INFOPATH"
```

各变量说明：

| 变量 | 作用 |
|------|------|
| `HOMEBREW_PREFIX` | Homebrew 安装根目录 |
| `PATH` | 将 Homebrew 的 `bin` 和 `sbin` 加入命令搜索路径，优先使用 Homebrew 安装的工具 |
| `HOMEBREW_CELLAR` | Homebrew 存放已安装软件包的目录 |
| `HOMEBREW_REPOSITORY` | Homebrew 核心仓库位置 |
| `MANPATH` | 手册页搜索路径 |
| `INFOPATH` | info 文档搜索路径 |

使配置生效：

```bash
source ~/.zshrc
```

### 1.3 验证安装

```bash
brew --version
brew doctor
```

`brew doctor` 会检查环境配置是否正确，按提示修复可能出现的警告即可。

## 二、安装 rbenv 管理 Ruby

[rbenv](https://github.com/rbenv/rbenv) 是轻量级的 Ruby 版本管理工具，配合 ruby-build 插件可以编译安装任意版本的 Ruby。

### 2.1 安装 rbenv 和 ruby-build

```bash
brew install rbenv ruby-build
```

`ruby-build` 是 rbenv 的插件，提供 `rbenv install` 命令来编译安装 Ruby。

### 2.2 配置 rbenv 环境

在 `~/.zshrc` 中添加：

```bash
# rbenv
eval "$(rbenv init - zsh)"
```

`rbenv init` 做了以下几件事：

- 将 `~/.rbenv/shims` 加入 `PATH` 最前面，使 rbenv 管理的 Ruby 版本优先于系统版本
- 安装 shell 函数用于 rehash（更新 shims）
- 启用自动补全

使配置生效：

```bash
source ~/.zshrc
```

### 2.3 安装 Ruby

```bash
# 查看可安装的版本
rbenv install -l

# 安装指定版本
rbenv install 3.3.7

# 设置全局默认版本
rbenv global 3.3.7

# 验证
ruby --version
```

`rbenv global` 会写入 `~/.rbenv/version` 文件，所有 shell 会话都会使用该版本。如果某个项目需要特定版本，可以在项目目录下执行 `rbenv local 3.x.x`，这会创建 `.ruby-version` 文件。

## 三、安装 pyenv 管理 Python

[pyenv](https://github.com/pyenv/pyenv) 的设计理念与 rbenv 类似，用于管理多个 Python 版本。

### 3.1 安装 pyenv

```bash
brew install pyenv
```

### 3.2 配置 pyenv 环境

在 `~/.zshrc` 中添加：

```bash
# pyenv
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"
```

各配置说明：

| 配置 | 作用 |
|------|------|
| `PYENV_ROOT` | pyenv 的安装根目录，所有 Python 版本都存放在这里 |
| `PATH` 追加 | 确保 `pyenv` 命令本身可被找到 |
| `pyenv init` | 将 `~/.pyenv/shims` 加入 PATH，启用版本切换和自动补全 |

使配置生效：

```bash
source ~/.zshrc
```

### 3.3 安装 Python 编译依赖

Python 从源码编译时需要一些系统库：

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

### 3.4 安装 Python

```bash
# 查看可安装的版本
pyenv install -l | grep -E "^\s+3\."

# 安装指定版本
pyenv install 3.13.3

# 设置全局默认版本
pyenv global 3.13.3

# 验证
python --version
pip --version
```

与 rbenv 类似，`pyenv global` 写入 `~/.pyenv/version`，`pyenv local` 在项目目录创建 `.python-version` 文件。

## 四、安装 nvm 管理 Node.js

[nvm](https://github.com/nvm-sh/nvm)（Node Version Manager）是 Node.js 的版本管理工具。与 rbenv/pyenv 不同，nvm 不通过 Homebrew 安装（官方不推荐），而是使用安装脚本。

### 4.1 安装 nvm

```bash
# 使用官方安装脚本
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
```

安装脚本会将 nvm 下载到 `~/.nvm` 目录，并自动在 `~/.zshrc` 中追加初始化配置。

### 4.2 确认环境配置

检查 `~/.zshrc` 中是否已添加以下内容（安装脚本通常会自动添加）：

```bash
# nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

| 配置 | 作用 |
|------|------|
| `NVM_DIR` | nvm 的安装目录 |
| `nvm.sh` | 加载 nvm 函数到当前 shell（nvm 是纯 shell 函数，不是二进制） |
| `bash_completion` | 启用 tab 自动补全 |

使配置生效：

```bash
source ~/.zshrc
```

### 4.3 安装 Node.js

```bash
# 查看可用的 LTS 版本
nvm ls-remote --lts

# 安装最新 LTS 版本
nvm install --lts

# 或安装指定版本
nvm install 22

# 设置默认版本
nvm alias default 22

# 验证
node --version
npm --version
```

`nvm alias default` 设置每次打开新终端时自动使用的 Node.js 版本。

## 五、完整的 ~/.zshrc 配置汇总

以下是本文涉及的所有环境配置，建议按此顺序添加到 `~/.zshrc`：

```bash
# Homebrew（用户目录安装）
export HOMEBREW_PREFIX="$HOME/.homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"
export HOMEBREW_CELLAR="$HOMEBREW_PREFIX/Cellar"
export HOMEBREW_REPOSITORY="$HOMEBREW_PREFIX"
export MANPATH="$HOMEBREW_PREFIX/share/man:$MANPATH"
export INFOPATH="$HOMEBREW_PREFIX/share/info:$INFOPATH"

# rbenv（Ruby 版本管理）
eval "$(rbenv init - zsh)"

# pyenv（Python 版本管理）
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"

# nvm（Node.js 版本管理）
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

## 六、注意事项

### 架构相关

- Apple Silicon Mac 使用 arm64 架构，部分旧版本的 Ruby/Python/Node.js 可能没有 arm64 预编译包，需要从源码编译，耗时会更长
- 如果遇到编译失败，优先检查是否安装了 Xcode Command Line Tools：`xcode-select --install`
- 某些依赖库可能需要指定架构标志，如遇到问题可尝试 `arch -arm64 brew install <package>`

### Homebrew 用户目录安装

- 用户目录安装的 Homebrew 不在官方支持的路径下，少数 formula 的 bottle（预编译包）可能不可用，会自动回退到源码编译
- 定期执行 `brew update && brew upgrade` 保持软件包更新
- 如果 `brew doctor` 报告警告，通常按提示操作即可

### 版本管理器通用建议

- `global` 设置影响所有终端会话，`local` 设置仅影响当前目录及子目录
- 项目中建议使用 `local` 并将版本文件（`.ruby-version`、`.python-version`、`.nvmrc`）提交到版本控制，确保团队成员使用一致的运行时版本
- 安装新版本后如果命令找不到，尝试执行 `rbenv rehash` 或重新打开终端
- 不建议使用 `sudo` 执行 `gem install`、`pip install` 或 `npm install -g`，版本管理器已经将这些路径指向用户空间

### Shell 配置加载顺序

`~/.zshrc` 中的配置顺序很重要：

1. Homebrew 的 PATH 配置应最先加载，因为 rbenv 和 pyenv 都是通过 Homebrew 安装的
2. rbenv/pyenv 的 `init` 会在 PATH 最前面插入 shims 目录，确保版本管理器的优先级高于 Homebrew 安装的系统版本
3. nvm 最后加载即可，它独立于 Homebrew

### 常用排查命令

```bash
# 查看当前使用的版本和来源
which ruby && ruby --version
which python && python --version
which node && node --version

# 查看版本管理器的版本切换状态
rbenv versions
pyenv versions
nvm ls

# 查看 PATH 中的搜索顺序
echo $PATH | tr ':' '\n'
```
