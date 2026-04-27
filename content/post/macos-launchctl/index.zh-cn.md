---
title: "macOS launchctl 详解：系统服务管理的核心工具"
description: "深入介绍 macOS 上 launchd/launchctl 的背景、原理、plist 配置与日常使用，涵盖定时任务、开机启动、常见问题排查等实用场景"
date: 2026-04-27T10:00:00+08:00
slug: macos-launchctl
image: "cover.svg"
categories:
    - 技术
tags:
    - macOS
    - launchctl
    - launchd
---

在 macOS 上，无论是开机自启服务、定时执行脚本，还是管理后台守护进程，背后都离不开 `launchd` 和它的命令行工具 `launchctl`。本文从背景、原理到实际配置，系统梳理 launchctl 的使用方式。

<!--more-->

## 背景

### 从 cron 到 launchd

早期 macOS（Mac OS X）沿用了 Unix 传统的 `cron` 和 `init` 来管理定时任务和系统服务。从 Mac OS X 10.4 Tiger 开始，Apple 引入了 `launchd` 作为 PID 1 进程，统一接管了以下职责：

- 系统初始化（替代 `init`）
- 服务管理（替代 `SystemStarter`）
- 定时任务（替代 `cron`）
- 文件监听触发（替代 `watchdog` 类工具）

`launchctl` 是与 `launchd` 交互的命令行工具，所有对服务的加载、卸载、启动、停止操作都通过它完成。

### 与 Linux systemd 的对比

如果你熟悉 Linux，可以这样类比：

| 概念 | macOS | Linux |
|------|-------|-------|
| PID 1 进程 | `launchd` | `systemd` |
| 管理工具 | `launchctl` | `systemctl` |
| 配置格式 | plist (XML) | unit 文件 (INI) |
| 配置目录 | 多个（按域区分） | `/etc/systemd/`、`~/.config/systemd/` |

## 核心概念

### Domain（域）

launchd 将服务按作用范围划分为不同的域：

- **system**：系统级服务，开机即运行，不依赖用户登录
- **gui/uid**：用户级服务，用户登录后运行（`gui/501` 表示 UID 为 501 的用户）
- **user/uid**：用户级后台服务

日常使用中最常接触的是 `system` 和 `gui/<uid>` 两个域。

### Service（服务）

每个服务由一个 plist 文件定义，包含服务的标识符（Label）、要执行的程序、运行条件等信息。服务有以下几种状态：

- **not loaded**：未加载，launchd 不知道这个服务
- **loaded / not running**：已加载但未运行，等待触发条件
- **running**：正在运行

### plist 配置目录

不同目录下的 plist 文件对应不同的用途和权限：

| 目录 | 域 | 用途 |
|------|------|------|
| `~/Library/LaunchAgents/` | 当前用户 | 用户自定义的登录后服务 |
| `/Library/LaunchAgents/` | 所有用户 | 管理员为所有用户配置的服务 |
| `/Library/LaunchDaemons/` | 系统 | 管理员配置的系统级守护进程 |
| `/System/Library/LaunchAgents/` | 所有用户 | Apple 提供的用户级服务（勿修改） |
| `/System/Library/LaunchDaemons/` | 系统 | Apple 提供的系统级服务（勿修改） |

日常使用中，我们主要在 `~/Library/LaunchAgents/` 下创建自己的服务配置。

> **Agent vs Daemon**：LaunchAgent 在用户登录后运行，可以访问 GUI 环境；LaunchDaemon 在系统启动时运行，以 root 身份执行，无法访问用户界面。

## plist 配置详解

plist（Property List）是 macOS 上的标准配置格式，launchd 使用 XML 格式的 plist 文件来定义服务。

### 最小配置

一个最简单的 plist 只需要 `Label` 和 `Program`（或 `ProgramArguments`）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.hello</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/echo</string>
        <string>Hello from launchd</string>
    </array>
</dict>
</plist>
```

### 常用配置项

| 键 | 类型 | 说明 |
|----|------|------|
| `Label` | String | 服务唯一标识符，通常使用反向域名格式 |
| `Program` | String | 要执行的程序路径 |
| `ProgramArguments` | Array | 程序及其参数（第一个元素为程序路径） |
| `RunAtLoad` | Boolean | 加载后立即运行一次 |
| `KeepAlive` | Boolean/Dict | 保持服务持续运行，退出后自动重启 |
| `StartInterval` | Integer | 每隔 N 秒运行一次 |
| `StartCalendarInterval` | Dict/Array | 按日历时间运行（类似 cron） |
| `WatchPaths` | Array | 监听文件路径变化时触发运行 |
| `StandardOutPath` | String | 标准输出重定向到文件 |
| `StandardErrorPath` | String | 标准错误重定向到文件 |
| `WorkingDirectory` | String | 工作目录 |
| `EnvironmentVariables` | Dict | 环境变量 |
| `UserName` | String | 以指定用户身份运行（仅 Daemon） |
| `ProcessType` | String | 进程类型，影响资源调度优先级 |

### 定时任务配置

`StartCalendarInterval` 的用法类似 cron，支持以下字段：

| 字段 | 取值范围 |
|------|----------|
| `Month` | 1–12 |
| `Day` | 1–31 |
| `Weekday` | 0–7（0 和 7 都表示周日） |
| `Hour` | 0–23 |
| `Minute` | 0–59 |

省略的字段表示"任意值"，与 cron 中的 `*` 等价。

每天凌晨 2:30 执行备份脚本：

```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>2</integer>
    <key>Minute</key>
    <integer>30</integer>
</dict>
```

每周一和周五的 9:00 执行：

```xml
<key>StartCalendarInterval</key>
<array>
    <dict>
        <key>Weekday</key>
        <integer>1</integer>
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <dict>
        <key>Weekday</key>
        <integer>5</integer>
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
</array>
```

### KeepAlive 高级配置

`KeepAlive` 除了简单的 `true/false`，还支持条件化配置：

```xml
<key>KeepAlive</key>
<dict>
    <key>SuccessfulExit</key>
    <false/>
    <key>NetworkState</key>
    <true/>
</dict>
```

上面的配置表示：仅在程序非正常退出（退出码非 0）且网络可用时才自动重启。

支持的条件：

| 条件 | 说明 |
|------|------|
| `SuccessfulExit` | `true`：正常退出后重启；`false`：异常退出后重启 |
| `NetworkState` | `true`：网络可用时保持运行 |
| `PathState` | 指定路径存在/不存在时保持运行 |
| `Crashed` | `true`：仅在崩溃时重启 |

## 实战示例

### 示例一：定时清理临时文件

创建 `~/Library/LaunchAgents/com.example.cleanup-tmp.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.cleanup-tmp</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/find</string>
        <string>/tmp/myapp</string>
        <string>-type</string>
        <string>f</string>
        <string>-mtime</string>
        <string>+7</string>
        <string>-delete</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/cleanup-tmp.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/cleanup-tmp.err</string>
</dict>
</plist>
```

### 示例二：开机启动的后台服务

创建一个登录后自动启动、崩溃自动重启的服务：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.myservice</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myservice</string>
        <string>--config</string>
        <string>/etc/myservice/config.yaml</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>Crashed</key>
        <true/>
    </dict>
    <key>WorkingDirectory</key>
    <string>/var/lib/myservice</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>LOG_LEVEL</key>
        <string>info</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/var/log/myservice/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/myservice/stderr.log</string>
</dict>
</plist>
```

### 示例三：监听文件变化触发构建

当配置文件发生变化时自动触发重新构建：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.config-watcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/rebuild.sh</string>
    </array>
    <key>WatchPaths</key>
    <array>
        <string>/etc/myapp/config.yaml</string>
    </array>
    <key>StandardOutPath</key>
    <string>/tmp/config-watcher.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/config-watcher.err</string>
</dict>
</plist>
```

## launchctl 命令使用

### 新版命令（macOS 10.10+）

从 Yosemite 开始，Apple 引入了基于子命令的新语法：

```bash
# 加载服务
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.myservice.plist

# 卸载服务
launchctl bootout gui/$(id -u)/com.example.myservice

# 启动服务（手动触发一次）
launchctl kickstart gui/$(id -u)/com.example.myservice

# 强制重启服务（-k 表示先杀掉再启动）
launchctl kickstart -k gui/$(id -u)/com.example.myservice

# 停止服务
launchctl kill SIGTERM gui/$(id -u)/com.example.myservice

# 查看服务状态
launchctl print gui/$(id -u)/com.example.myservice

# 列出某个域下的所有服务
launchctl print gui/$(id -u)

# 禁用服务（重启后也不会加载）
launchctl disable gui/$(id -u)/com.example.myservice

# 启用服务
launchctl enable gui/$(id -u)/com.example.myservice
```

### 旧版命令（仍然可用）

旧版命令更简洁，很多教程和脚本仍在使用：

```bash
# 加载服务
launchctl load ~/Library/LaunchAgents/com.example.myservice.plist

# 加载并强制覆盖 disabled 状态
launchctl load -w ~/Library/LaunchAgents/com.example.myservice.plist

# 卸载服务
launchctl unload ~/Library/LaunchAgents/com.example.myservice.plist

# 卸载并标记为 disabled
launchctl unload -w ~/Library/LaunchAgents/com.example.myservice.plist

# 列出所有已加载的服务
launchctl list

# 查看某个服务的信息
launchctl list com.example.myservice

# 手动启动
launchctl start com.example.myservice

# 手动停止
launchctl stop com.example.myservice
```

> 新版命令和旧版命令可以混用，但建议在新脚本中使用新版语法，因为它提供了更精确的域控制和更丰富的功能。

### 常用操作速查

| 操作 | 新版命令 | 旧版命令 |
|------|----------|----------|
| 加载 | `launchctl bootstrap gui/$(id -u) <plist>` | `launchctl load <plist>` |
| 卸载 | `launchctl bootout gui/$(id -u)/<label>` | `launchctl unload <plist>` |
| 启动 | `launchctl kickstart gui/$(id -u)/<label>` | `launchctl start <label>` |
| 停止 | `launchctl kill SIGTERM gui/$(id -u)/<label>` | `launchctl stop <label>` |
| 查看状态 | `launchctl print gui/$(id -u)/<label>` | `launchctl list <label>` |
| 禁用 | `launchctl disable gui/$(id -u)/<label>` | `launchctl unload -w <plist>` |

## 调试与排查

### 查看服务日志

首先检查你在 plist 中配置的 `StandardOutPath` 和 `StandardErrorPath` 指向的日志文件。

如果没有配置输出路径，可以查看系统日志：

```bash
# 使用 log 命令查看 launchd 相关日志
log show --predicate 'process == "launchd"' --last 1h

# 过滤特定服务的日志
log show --predicate 'subsystem == "com.example.myservice"' --last 1h
```

### 检查 plist 语法

```bash
# 验证 plist 格式是否正确
plutil -lint ~/Library/LaunchAgents/com.example.myservice.plist

# 将 plist 转换为可读的 XML 格式查看
plutil -convert xml1 -o - ~/Library/LaunchAgents/com.example.myservice.plist
```

### 查看退出状态码

```bash
launchctl list com.example.myservice
```

输出中的 `"LastExitStatus"` 字段会显示上次退出的状态码，非 0 表示异常退出。

## 常见问题与注意事项

### 1. 服务加载后不运行

- 检查是否设置了 `RunAtLoad` 为 `true`，否则服务只会在触发条件满足时运行
- 确认程序路径是绝对路径，launchd 不会使用 shell 的 `PATH` 环境变量
- 用 `plutil -lint` 检查 plist 语法

### 2. 脚本执行没有效果

launchd 启动的进程不会加载用户的 shell 配置（`.bashrc`、`.zshrc` 等），因此：

- 所有命令必须使用绝对路径（如 `/usr/local/bin/python3` 而非 `python3`）
- 需要的环境变量必须在 plist 的 `EnvironmentVariables` 中显式声明
- 如果必须使用 shell 环境，可以将 `ProgramArguments` 设置为通过 shell 执行：

```xml
<key>ProgramArguments</key>
<array>
    <string>/bin/zsh</string>
    <string>-l</string>
    <string>-c</string>
    <string>/path/to/your/script.sh</string>
</array>
```

### 3. 权限问题

- `~/Library/LaunchAgents/` 下的 plist 文件权限应为 `644`
- `/Library/LaunchDaemons/` 下的 plist 文件所有者必须是 `root:wheel`，权限 `644`
- 如果权限不对，launchd 会拒绝加载并在日志中报错

```bash
# 修复用户 Agent 权限
chmod 644 ~/Library/LaunchAgents/com.example.myservice.plist

# 修复系统 Daemon 权限
sudo chown root:wheel /Library/LaunchDaemons/com.example.myservice.plist
sudo chmod 644 /Library/LaunchDaemons/com.example.myservice.plist
```

### 4. 定时任务错过执行时间

如果 Mac 在计划执行时间处于睡眠或关机状态，`StartCalendarInterval` 定义的任务会在唤醒/开机后立即补执行一次（仅补一次，不会补执行所有错过的次数）。

### 5. 服务反复重启

如果设置了 `KeepAlive` 为 `true`，但程序启动后立即退出，launchd 会不断尝试重启。launchd 内置了节流机制：如果服务在 10 秒内退出，会等待 10 秒后再重启。

如果看到 `"Throttling respawn"` 相关日志，说明程序存在启动失败的问题，应该先修复程序本身。

### 6. 修改 plist 后不生效

修改 plist 文件后必须重新加载才能生效：

```bash
# 先卸载旧配置
launchctl bootout gui/$(id -u)/com.example.myservice

# 再加载新配置
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.myservice.plist
```

直接编辑已加载的 plist 文件不会自动生效，launchd 只在加载时读取配置。

### 7. SIP 对 /System 目录的保护

macOS 的系统完整性保护（SIP）会阻止修改 `/System/Library/LaunchAgents/` 和 `/System/Library/LaunchDaemons/` 下的文件。不要尝试修改这些目录中的系统服务配置，如需自定义系统级服务，请使用 `/Library/LaunchDaemons/`。

## 实用技巧

### 使用 PlistBuddy 编辑 plist

除了手动编辑 XML，还可以使用 `PlistBuddy` 命令行工具：

```bash
# 读取某个键的值
/usr/libexec/PlistBuddy -c "Print :Label" ~/Library/LaunchAgents/com.example.myservice.plist

# 修改某个键的值
/usr/libexec/PlistBuddy -c "Set :StartInterval 600" ~/Library/LaunchAgents/com.example.myservice.plist

# 添加新键
/usr/libexec/PlistBuddy -c "Add :RunAtLoad bool true" ~/Library/LaunchAgents/com.example.myservice.plist
```

### 快速查看所有用户服务

```bash
# 列出所有用户级服务及其状态
launchctl list | grep -v "com.apple"
```

### 使用 Homebrew 管理的服务

如果你使用 Homebrew，`brew services` 命令是对 launchctl 的封装，可以更方便地管理通过 Homebrew 安装的服务：

```bash
# 列出所有 Homebrew 服务
brew services list

# 启动服务
brew services start mysql

# 停止服务
brew services stop mysql

# 重启服务
brew services restart mysql
```

`brew services` 会自动在 `~/Library/LaunchAgents/` 下生成和管理 plist 文件。

## 参考资料

- [Apple Developer - launchd.plist man page](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
- `man launchd.plist` — 完整的 plist 配置项说明
- `man launchctl` — launchctl 命令参考
