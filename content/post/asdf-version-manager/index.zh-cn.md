---
title: "使用 asdf 统一管理开发语言版本"
description: "在 macOS ARM64 环境下安装配置 asdf，统一管理 Ruby、Python、Node.js、Golang、Java 等多语言运行时版本"
date: 2026-04-25T10:00:00+08:00
slug: asdf-version-manager
image: "cover.svg"
categories:
    - 技术
tags:
    - macOS
    - asdf
    - Ruby
    - Python
    - Node.js
    - Golang
    - Java
---

在[上一篇文章](/p/macos-dev-env-setup/)中，我们分别使用 rbenv、pyenv、nvm 来管理 Ruby、Python、Node.js 的版本。这种方式可以工作，但每个工具有各自的安装方式、配置语法和命令风格，管理成本随语言数量线性增长。

[asdf](https://asdf-vm.com/) 是一个可扩展的版本管理器，通过统一的命令接口和插件体系，用一个工具管理几乎所有语言和工具的版本。本文记录在 macOS arm64 环境下安装 asdf，并配置 Ruby、Python、Node.js、Golang、Java 的完整过程。

<!--more-->

## 为什么选择 asdf

相比为每种语言分别安装版本管理器，asdf 有几个明显优势：

- 统一命令体系，学习成本更低，比如都使用 `asdf install`、`asdf set`、`asdf current`
- 统一配置文件，项目根目录下一个 `.tool-versions` 就可以声明多个语言版本
- 插件生态丰富，不止支持编程语言，也支持很多常用 CLI 工具
- 更适合多语言项目，比如一个仓库同时依赖 Node.js、Ruby、Java、Go
- 团队协作更直接，提交 `.tool-versions` 后，其他成员执行 `asdf install` 即可对齐环境

它也不是完全没有代价。asdf 本质上是一个通用框架，很多具体语言能力来自插件，所以不同插件的质量、维护状态和细节体验并不完全一致。但从整体工程效率来看，对于需要同时维护多种运行时的人来说，asdf 通常比多套版本管理器更省心。

## 一、前置准备

### 1.1 安装 Xcode Command Line Tools

很多语言版本在 Apple Silicon 上需要源码编译，因此先安装命令行开发工具：

```bash
xcode-select --install
```

如果已经安装，会收到提示，可以直接跳过。

### 1.2 安装 Homebrew

本文默认你已经安装 Homebrew。若还没有安装，可以参考我之前写的 [macOS ARM64 开发环境搭建指南](/p/macos-dev-env-setup/)。

先确认 Homebrew 可用：

```bash
brew --version
```

### 1.3 安装常见编译依赖

Ruby、Python、Java 等运行时在安装过程中经常会依赖系统库，建议先装好一批常用基础包：

```bash
brew install coreutils curl git openssl readline sqlite3 xz zlib tcl-tk libyaml gmp autoconf automake libtool
```

不是每种语言都会用到这些库，但提前准备可以减少后续编译失败的概率。

## 二、安装 asdf

### 2.1 使用 Homebrew 安装

在 macOS 上，最直接的方式是通过 Homebrew 安装：

```bash
brew install asdf
```

安装完成后确认版本：

```bash
asdf --version
```

### 2.2 配置 shell 环境

如果你使用的是 zsh，在 `~/.zshrc` 中加入：

```bash
# asdf
export ASDF_DATA_DIR="$HOME/.asdf"
export PATH="$ASDF_DATA_DIR/shims:$PATH"
```

然后重新加载配置：

```bash
source ~/.zshrc
```

这里需要注意两点：

- `ASDF_DATA_DIR` 指定 asdf 的数据目录，插件、已安装版本、shim 都在这里
- `shims` 必须在 `PATH` 中，这样 `ruby`、`python`、`node`、`go`、`java` 等命令才会被 asdf 接管

有些安装方式会推荐执行 `asdf init` 相关命令，但在当前版本的 asdf + Homebrew 组合下，通常只要保证可执行文件和 shims 在 PATH 中即可正常工作。

### 2.3 验证安装

```bash
which asdf
asdf info
```

`asdf info` 会输出当前操作系统、shell、asdf 路径等信息，排查环境问题时很有用。

## 三、asdf 的基本工作方式

在安装具体语言之前，先理解几个核心概念。

### 3.1 插件

asdf 本身只负责版本切换和分发逻辑，具体安装某种语言要靠插件。

例如：

- `ruby` 对应 Ruby 插件
- `python` 对应 Python 插件
- `nodejs` 对应 Node.js 插件
- `golang` 对应 Go 插件
- `java` 对应 Java 插件

### 3.2 版本范围

asdf 一般有三种生效范围：

- 全局版本：对当前用户的大多数 shell 生效
- 本地版本：仅对当前目录及其子目录生效
- 临时 shell 版本：仅对当前 shell 会话生效

### 3.3 `.tool-versions`

asdf 最重要的配置文件是 `.tool-versions`。例如：

```text
ruby 3.3.7
python 3.13.3
nodejs 22.14.0
golang 1.24.2
java temurin-21.0.7+6.0.LTS
```

这个文件通常放在：

- 用户家目录，用于全局默认版本
- 项目根目录，用于项目级版本固定

当你进入某个目录时，asdf 会向上查找 `.tool-versions`，决定当前应该使用哪个版本。

### 3.4 常用命令速览

```bash
# 添加插件
asdf plugin add <name>

# 查看插件列表
asdf plugin list

# 查看某语言可安装版本
asdf list all <name>

# 安装指定版本
asdf install <name> <version>

# 设置当前目录版本
asdf set <name> <version>

# 设置全局版本
asdf set -u <name> <version>

# 查看当前生效版本
asdf current

# 查看已安装版本
asdf list <name>
```

其中：

- `asdf set <name> <version>` 会在当前目录写入 `.tool-versions`
- `asdf set -u <name> <version>` 会写入用户级别配置

## 四、安装和管理 Ruby

### 4.1 安装 Ruby 插件

```bash
asdf plugin add ruby
```

确认插件已安装：

```bash
asdf plugin list
```

### 4.2 安装 Ruby 编译依赖

在 macOS ARM64 上，Ruby 编译经常依赖 OpenSSL、libyaml、readline 等库：

```bash
brew install openssl readline libyaml gmp
```

### 4.3 查看并安装 Ruby 版本

```bash
# 查看可安装版本
asdf list all ruby

# 安装指定版本
asdf install ruby 3.3.7
```

如果安装较慢，这是正常现象。Ruby 往往需要源码编译。

### 4.4 设置全局或项目版本

设置全局默认版本：

```bash
asdf set -u ruby 3.3.7
```

设置当前项目版本：

```bash
asdf set ruby 3.3.7
```

验证：

```bash
ruby --version
which ruby
asdf current ruby
```

### 4.5 Bundler 和 gem 的使用建议

安装完成 Ruby 后，建议更新 RubyGems 和 Bundler：

```bash
gem update --system
gem install bundler
```

在 asdf 管理下，`gem install` 默认落在当前 Ruby 版本自己的目录中，一般不需要 `sudo`。

## 五、安装和管理 Python

### 5.1 安装 Python 插件

```bash
asdf plugin add python
```

### 5.2 安装 Python 编译依赖

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

### 5.3 安装 Python

```bash
# 查看可安装版本
asdf list all python

# 安装指定版本
asdf install python 3.13.3
```

### 5.4 设置版本并验证

```bash
asdf set -u python 3.13.3
python --version
pip --version
which python
asdf current python
```

如果是项目级 Python 版本，则在项目目录执行：

```bash
asdf set python 3.13.3
```

### 5.5 Python 虚拟环境建议

asdf 负责的是 Python 解释器版本，不负责替代虚拟环境工具。实际项目中依然建议配合：

- `python -m venv`
- `poetry`
- `uv`
- `pipenv`

也就是说：

- asdf 解决“用哪个 Python 版本”
- 虚拟环境工具解决“当前项目依赖装在哪里”

这两个层次不要混淆。

## 六、安装和管理 Node.js

### 6.1 安装 Node.js 插件

```bash
asdf plugin add nodejs
```

### 6.2 导入 Node.js 发布密钥

Node.js 插件通常需要导入发布团队的 GPG key，用于校验下载内容。先执行：

```bash
bash ~/.asdf/plugins/nodejs/bin/import-release-team-keyring
```

如果你是第一次安装 Node.js，这一步不要跳过。

### 6.3 安装 Node.js

```bash
# 查看所有可安装版本
asdf list all nodejs

# 安装指定版本
asdf install nodejs 22.14.0
```

### 6.4 设置版本并验证

```bash
asdf set -u nodejs 22.14.0
node --version
npm --version
which node
asdf current nodejs
```

如果项目中已经有 `.nvmrc`，可以手动将它对应的版本同步到 `.tool-versions`，逐步从 nvm 迁移到 asdf。

### 6.5 Corepack 建议

如果项目使用 `pnpm` 或 `yarn`，建议启用 Corepack：

```bash
corepack enable
```

这样包管理器版本也能更稳定地和 Node.js 运行时配合。

## 七、安装和管理 Golang

### 7.1 安装 Go 插件

```bash
asdf plugin add golang
```

### 7.2 安装 Go

```bash
# 查看可安装版本
asdf list all golang

# 安装指定版本
asdf install golang 1.24.2
```

### 7.3 设置版本并验证

```bash
asdf set -u golang 1.24.2
go version
which go
asdf current golang
```

### 7.4 GOPATH 与 GOBIN 建议

现代 Go 项目通常不需要手动设置复杂的 `GOPATH`，但为了兼容一些工具，可以在 `~/.zshrc` 中加上：

```bash
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"
```

这样 `go install` 安装的命令行工具可以直接使用。

## 八、安装和管理 Java

### 8.1 安装 Java 插件

```bash
asdf plugin add java
```

### 8.2 查看 Java 发行版和版本

asdf 的 Java 插件支持多个发行版，例如：

- temurin
- corretto
- zulu
- openjdk

先查看可安装版本：

```bash
asdf list all java
```

输出通常会比较长，建议按关键字过滤：

```bash
asdf list all java | grep temurin
```

### 8.3 安装 Java

例如安装 Temurin 21：

```bash
asdf install java temurin-21.0.7+6.0.LTS
```

### 8.4 设置版本并验证

```bash
asdf set -u java temurin-21.0.7+6.0.LTS
java -version
which java
asdf current java
```

### 8.5 `JAVA_HOME` 配置

很多 Java 工具链仍然依赖 `JAVA_HOME`。可以在 `~/.zshrc` 中加入：

```bash
export JAVA_HOME="$HOME/.asdf/installs/java/$(asdf current java | awk '{print $2}')"
```

不过这类写法依赖 shell 启动时已能正确执行 `asdf`。更稳妥的做法是：

- 将 `java` 版本固定在 `.tool-versions`
- 在具体项目脚本、IDE 或构建工具中按项目配置 `JAVA_HOME`

对于 IntelliJ IDEA 之类的 IDE，通常直接在 IDE 的 JDK 设置中选择 asdf 安装目录下的具体版本更清晰。

## 九、完整示例：为一个多语言项目配置 `.tool-versions`

假设一个项目同时包含：

- Rails 后端
- Python 数据脚本
- Node.js 前端构建
- Go 工具
- Java 服务

那么项目根目录下的 `.tool-versions` 可以写成：

```text
ruby 3.3.7
python 3.13.3
nodejs 22.14.0
golang 1.24.2
java temurin-21.0.7+6.0.LTS
```

然后执行：

```bash
asdf install
```

asdf 会根据 `.tool-versions` 自动安装缺失版本。这个体验是 asdf 最有价值的地方之一：环境声明与环境安装天然绑定。

## 十、常用排查方法

### 10.1 命令版本不对

先看当前 asdf 识别到的版本：

```bash
asdf current
```

再看实际命令路径：

```bash
which ruby
which python
which node
which go
which java
```

如果 `which` 没有指向 `~/.asdf/shims/...`，通常说明 PATH 顺序不对。

### 10.2 插件正常但命令找不到

可以尝试刷新 shim：

```bash
asdf reshim
```

例如在安装某些 Ruby gem 或 Node.js 全局命令后，reshim 很有帮助。

### 10.3 安装失败

先看三个常见原因：

- Xcode Command Line Tools 没装好
- Homebrew 依赖缺失
- Apple Silicon 下某个旧版本本身兼容性较差

此时优先执行：

```bash
asdf info
brew doctor
xcode-select -p
```

### 10.4 某些旧版本在 ARM64 上编译失败

这是 Apple Silicon 上比较常见的问题，尤其是很老的 Ruby、Python、Node.js 版本。排查思路是：

- 优先使用仍在维护周期内的新版本
- 查对应插件仓库的 issue
- 必要时为特定版本补充编译参数
- 实在不兼容时，再考虑通过 Rosetta 运行单独的 x86_64 环境

一般不建议默认把整个终端都切到 Rosetta，只为兼容极少数旧版本工具。

## 十一、从 rbenv / pyenv / nvm 迁移到 asdf 的建议

如果你当前机器上已经有 rbenv、pyenv、nvm，不建议一次性粗暴删除。更稳妥的迁移方式是：

1. 先安装 asdf 和所需插件
2. 用 asdf 安装与你当前项目一致的语言版本
3. 在项目目录创建 `.tool-versions`
4. 打开一个全新 shell，确认 `ruby`、`python`、`node` 等命令已由 asdf 接管
5. 确认没有问题后，再逐步移除旧版本管理器的初始化配置

迁移时最容易出问题的是 PATH 顺序。只要多个版本管理器都在改 PATH，就可能出现“安装成功但实际没生效”的情况。

## 十二、推荐的 `~/.zshrc` 配置示例

如果你主要依赖 Homebrew 和 asdf，可以参考下面这份精简配置：

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

如果你的 Homebrew 安装在用户目录，则将 `HOMEBREW_PREFIX` 改成你自己的安装路径即可。

## 十三、总结

对于 macOS ARM64 开发环境，asdf 的价值不在于“它能装更多语言”，而在于它把版本管理这件事统一成了一套稳定的工作流：添加插件、安装版本、写入 `.tool-versions`、进入目录自动切换。

当项目涉及 Ruby、Python、Node.js、Go、Java 多种运行时时，这种统一性会明显降低环境维护成本。尤其在团队协作和新机器初始化场景下，asdf 往往比多套版本管理器更容易长期维护。
