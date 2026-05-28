---
title: "Protobuf 自动生成流程最佳实践"
description: "规范化管理 .proto 文件，实现多语言代码自动生成与 CI 集成"
date: 2026-05-15T10:00:00+08:00
slug: protobuf-auto-generation
image: "cover.svg"
categories:
    - 技术
tags:
    - Protobuf
    - gRPC
    - CI/CD
---

在多端协作的项目中，Protobuf 作为跨语言的数据结构定义与序列化方案，能够有效统一移动端、服务端、嵌入式端之间的通信协议。本文整理了一套适合团队协作的 Protobuf 自动生成流程，涵盖目录规范、工具链选型、CI 集成以及常见问题处理，帮助团队降低手动维护生成代码的风险。

<!--more-->

## 一、为什么需要自动化流程

在实际项目中，Protobuf 的价值不仅在于定义数据结构，更在于保证多端协议一致性。手动维护生成代码容易出现以下问题：

- 不同开发者使用的 `protoc` 版本不一致，导致生成代码存在差异；
- 修改 `.proto` 后忘记重新生成代码，造成协议不同步；
- 多语言生成命令复杂，新人接入成本高；
- 缺少版本管理与兼容性检查，容易引入破坏性变更。

自动化流程的核心目标是：

1. **协议定义唯一化**：所有数据结构以 `.proto` 文件为唯一可信来源；
2. **生成流程自动化**：通过脚本或构建任务完成代码生成，避免手动操作；
3. **跨语言一致性**：多端生成代码来自同一套 `.proto` 文件；
4. **可追踪、可审计**：变更记录可被 Git 追溯，CI 自动检查生成代码是否同步。

## 二、推荐目录结构

一个规范的 Protobuf 项目目录应该清晰区分源文件、生成代码、脚本工具和 CI 配置：

```text
ProjectRoot/
├── Protos/
│   ├── common/
│   │   ├── timestamp.proto
│   │   └── error.proto
│   ├── vehicle/
│   │   ├── vehicle_command.proto
│   │   ├── vehicle_state.proto
│   │   └── vehicle_auth.proto
│   └── README.md
│
├── Generated/
│   ├── Swift/
│   │   └── Sources/
│   ├── Kotlin/
│   │   └── src/main/java/
│   ├── Go/
│   │   └── pb/
│   └── TypeScript/
│       └── src/generated/
│
├── Scripts/
│   ├── generate-protobuf.sh
│   ├── check-protobuf.sh
│   └── clean-generated.sh
│
├── Tools/
│   └── protobuf/
│       └── README.md
│
└── .github/
    └── workflows/
        └── protobuf-check.yml
```

目录说明：

| 目录 | 说明 |
|---|---|
| `Protos/` | 存放所有 `.proto` 源文件 |
| `Generated/` | 存放自动生成的目标语言代码 |
| `Scripts/` | 存放生成、检查、清理脚本 |
| `Tools/` | 记录 protobuf 工具安装方式和版本要求 |
| `.github/workflows/` | CI 中的自动检查流程 |

## 三、.proto 文件设计规范

### 文件命名

推荐使用小写下划线命名：

```text
vehicle_command.proto
vehicle_state.proto
vehicle_auth.proto
```

避免使用驼峰或大写开头的命名方式，保持与主流开源项目一致。

### package 命名

建议按业务域划分，采用 `公司名.业务域.模块名` 的层级结构：

```proto
syntax = "proto3";

package tesla.vehicle.command;
```

这种命名方式便于在大型项目中区分不同业务模块，避免命名冲突。

### 字段编号规范

字段编号一旦发布，不应随意修改或复用：

```proto
message VehicleCommand {
  string vehicle_id = 1;
  CommandType type = 2;
  bytes payload = 3;
}
```

关键规则：

- 字段编号必须稳定，删除字段后应使用 `reserved` 保留编号；
- 不要复用历史字段编号；
- 常用字段建议使用较小编号，提升序列化效率。

示例：

```proto
message VehicleCommand {
  reserved 4, 5;
  reserved "legacy_token", "old_signature";

  string vehicle_id = 1;
  CommandType type = 2;
  bytes payload = 3;
}
```

### 枚举规范

枚举必须提供默认值，通常使用 `UNSPECIFIED` 或 `UNKNOWN`：

```proto
enum CommandType {
  COMMAND_TYPE_UNSPECIFIED = 0;
  COMMAND_TYPE_LOCK = 1;
  COMMAND_TYPE_UNLOCK = 2;
  COMMAND_TYPE_WAKE = 3;
}
```

这样可以避免反序列化时出现未定义值的问题。

### 向后兼容原则

允许的变更：

- 新增字段
- 新增 enum 值
- 新增 message
- 新增 service 方法

高风险变更：

- 修改字段编号
- 修改字段类型
- 删除字段但不 reserved
- 修改 package 名称
- 修改已有 enum 数值

## 四、工具链选型

### 基础工具

需要安装的基础工具包括：

```text
protoc
protoc-gen-swift
protoc-gen-grpc-swift
protoc-gen-go
protoc-gen-go-grpc
protoc-gen-ts 或 buf
```

具体语言插件可按项目需要选择。

### 推荐方案：使用 Buf 管理

对于中大型项目，推荐使用 Buf 统一管理 Protobuf 工作流。Buf 的优势包括：

- 统一 lint 规则
- 支持 breaking change 检查
- 统一生成多语言代码
- 配置更清晰
- 更适合 CI 集成

核心配置文件：

**buf.yaml**

```yaml
version: v1
name: buf.build/example/vehicle
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT
```

**buf.gen.yaml**

```yaml
version: v1
plugins:
  - plugin: swift
    out: Generated/Swift/Sources
  - plugin: grpc-swift
    out: Generated/Swift/Sources
  - plugin: go
    out: Generated/Go
    opt:
      - paths=source_relative
  - plugin: go-grpc
    out: Generated/Go
    opt:
      - paths=source_relative
```

执行生成：

```bash
buf generate
```

执行 lint：

```bash
buf lint
```

执行 breaking change 检查：

```bash
buf breaking --against '.git#branch=main'
```

## 五、基于 protoc 的生成流程

如果项目规模较小或不希望引入 Buf，也可以直接使用 `protoc` 编写生成脚本。

### Swift 生成脚本示例

文件：`Scripts/generate-protobuf.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
PROTO_DIR="$ROOT_DIR/Protos"
SWIFT_OUT="$ROOT_DIR/Generated/Swift/Sources"

mkdir -p "$SWIFT_OUT"

PROTO_FILES=$(find "$PROTO_DIR" -name "*.proto")

protoc \
  --proto_path="$PROTO_DIR" \
  --swift_out="$SWIFT_OUT" \
  --grpc-swift_out="$SWIFT_OUT" \
  $PROTO_FILES

echo "protobuf Swift code generated successfully."
```

### 多语言统一生成脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
PROTO_DIR="$ROOT_DIR/Protos"
GENERATED_DIR="$ROOT_DIR/Generated"

PROTO_FILES=$(find "$PROTO_DIR" -name "*.proto")

mkdir -p "$GENERATED_DIR/Swift/Sources"
mkdir -p "$GENERATED_DIR/Go"
mkdir -p "$GENERATED_DIR/TypeScript/src/generated"

protoc \
  --proto_path="$PROTO_DIR" \
  --swift_out="$GENERATED_DIR/Swift/Sources" \
  --grpc-swift_out="$GENERATED_DIR/Swift/Sources" \
  $PROTO_FILES

protoc \
  --proto_path="$PROTO_DIR" \
  --go_out="$GENERATED_DIR/Go" \
  --go-grpc_out="$GENERATED_DIR/Go" \
  $PROTO_FILES

protoc \
  --proto_path="$PROTO_DIR" \
  --ts_out="$GENERATED_DIR/TypeScript/src/generated" \
  $PROTO_FILES

echo "all protobuf code generated successfully."
```

## 六、Swift Package 集成建议

如果项目中存在 Swift Package，推荐将生成代码放入：

```text
Sources/TeslaVehicleProtocol/Generated/
```

### Package.swift 示例

```swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "TeslaVehicleProtocol",
    platforms: [
        .iOS(.v15),
        .macOS(.v12)
    ],
    products: [
        .library(
            name: "TeslaVehicleProtocol",
            targets: ["TeslaVehicleProtocol"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.0.0"),
        .package(url: "https://github.com/grpc/grpc-swift.git", from: "1.0.0")
    ],
    targets: [
        .target(
            name: "TeslaVehicleProtocol",
            dependencies: [
                .product(name: "SwiftProtobuf", package: "swift-protobuf"),
                .product(name: "GRPC", package: "grpc-swift")
            ],
            path: "Sources/TeslaVehicleProtocol"
        )
    ]
)
```

## 七、CI 检查流程

### CI 目标

CI 应检查以下内容：

1. `.proto` 文件语法正确
2. protobuf lint 通过
3. 自动生成代码与仓库中代码一致
4. 不存在破坏兼容性的变更
5. Swift Package / Go Module / TypeScript 构建通过

### GitHub Actions 示例

文件：`.github/workflows/protobuf-check.yml`

```yaml
name: Protobuf Check

on:
  pull_request:
    paths:
      - "Protos/**"
      - "Generated/**"
      - "Scripts/**"
      - "buf.yaml"
      - "buf.gen.yaml"

jobs:
  protobuf-check:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Buf
        uses: bufbuild/buf-setup-action@v1

      - name: Lint protobuf
        run: buf lint

      - name: Generate protobuf code
        run: buf generate

      - name: Check generated code is up to date
        run: |
          if ! git diff --exit-code; then
            echo "Generated protobuf code is out of date."
            echo "Please run buf generate or ./Scripts/generate-protobuf.sh locally."
            exit 1
          fi

      - name: Check breaking changes
        run: buf breaking --against '.git#branch=main'
```

## 八、是否提交生成代码

### 推荐提交生成代码的情况

适合提交生成代码：

- 移动端项目
- SDK 项目
- Swift Package
- 多端协作项目
- CI 环境不希望安装复杂生成工具
- 使用方不希望自己生成 protobuf 代码

优点：

- 使用简单
- 构建稳定
- 代码审查时能看到生成结果
- 降低接入成本

缺点：

- Git diff 较大
- 生成代码可能造成代码审查噪音

### 不提交生成代码的情况

适合不提交生成代码：

- 服务端 mono repo
- CI 构建环境完全可控
- 生成流程非常稳定
- 使用 Bazel、Buck、Buf 等统一构建系统

优点：

- 仓库更干净
- 避免生成代码 diff 噪音

缺点：

- 构建环境要求更高
- 新人接入成本略高
- 工具版本不一致时容易出问题

### 推荐结论

对于 Swift SDK、移动端 App、跨端协议库，建议提交生成代码。  
对于服务端 mono repo，可考虑只提交 `.proto`，由 CI 或构建系统生成。

## 九、版本管理策略

### proto 文件版本

可以通过 package 或目录区分版本：

```text
Protos/vehicle/v1/vehicle_command.proto
Protos/vehicle/v2/vehicle_command.proto
```

对应 package：

```proto
package tesla.vehicle.v1;
```

### 字段演进策略

新增字段：

```proto
message VehicleState {
  string vehicle_id = 1;
  bool locked = 2;
  int32 battery_level = 3;
}
```

废弃字段：

```proto
message VehicleState {
  string vehicle_id = 1;
  bool locked = 2;

  reserved 3;
  reserved "battery_level";

  int32 battery_percentage = 4;
}
```

### API 兼容策略

推荐遵循：

```text
新增不破坏
删除需 reserved
改类型视为破坏
改字段编号禁止
改语义需新增字段
```

## 十、生成脚本最佳实践

### 脚本必须可重复执行

生成脚本应具备幂等性：

```bash
./Scripts/generate-protobuf.sh
./Scripts/generate-protobuf.sh
```

连续执行两次不应产生额外 diff。

### 生成前清理旧文件

```bash
rm -rf Generated/Swift/Sources/*
rm -rf Generated/Go/*
```

注意：只清理生成目录，不要误删手写代码。

### 固定工具版本

建议在文档或工具配置中明确版本：

```text
protoc: 25.x
swift-protobuf: 1.x
grpc-swift: 1.x
buf: 1.x
```

也可以使用版本管理工具：

- Homebrew Bundle
- Mint
- Makefile
- Docker
- mise
- asdf

## 十一、Makefile 示例

```makefile
.PHONY: proto proto-lint proto-check proto-clean

proto:
	./Scripts/generate-protobuf.sh

proto-lint:
	buf lint

proto-breaking:
	buf breaking --against '.git#branch=main'

proto-check: proto-lint proto
	git diff --exit-code

proto-clean:
	./Scripts/clean-generated.sh
```

使用方式：

```bash
make proto
make proto-check
```

## 十二、常见问题

### 生成代码每次都有 diff

可能原因：

- protoc 版本不一致
- 插件版本不一致
- 输出路径不稳定
- 文件排序不稳定
- 不同操作系统换行符不同

解决方式：

- 固定工具版本
- 使用 Buf 或 Docker 统一环境
- 在脚本中排序 proto 文件
- 配置 `.gitattributes`

示例：

```bash
PROTO_FILES=$(find "$PROTO_DIR" -name "*.proto" | sort)
```

### 新增字段后旧客户端是否能兼容

通常可以兼容。旧客户端会忽略未知字段，新客户端也应为缺失字段提供默认处理逻辑。

### 是否可以修改字段类型

不建议。修改字段类型可能导致反序列化异常或语义错误。更安全的做法是新增字段，并废弃旧字段。

### 删除字段后可以复用编号吗

不可以。应使用 `reserved` 保留历史编号和字段名。

### proto 文件是否需要代码审查

需要。`.proto` 是跨端协议契约,任何字段编号、类型、语义变化都可能影响多个端。

## 十三、推荐落地流程

### 初始接入

1. 建立 `Protos/` 目录
2. 建立 `Generated/` 目录
3. 编写 `buf.yaml` 和 `buf.gen.yaml`
4. 编写 `Scripts/generate-protobuf.sh`
5. 编写 `Makefile`
6. 接入 GitHub Actions
7. 在 README 中说明开发者使用方式

### 日常开发

```text
修改 .proto
    ↓
执行 make proto
    ↓
运行测试
    ↓
检查 git diff
    ↓
提交 proto + generated code
    ↓
CI 校验
    ↓
Code Review
    ↓
合并
```

### Code Review 检查清单

| 检查项 | 说明 |
|---|---|
| 字段编号是否稳定 | 是否误改已有字段编号 |
| 字段类型是否兼容 | 是否修改了已有字段类型 |
| 删除字段是否 reserved | 是否保留历史编号和字段名 |
| enum 是否有默认值 | 是否存在 `UNSPECIFIED = 0` |
| package 是否合理 | 是否符合业务域命名 |
| 生成代码是否同步 | 是否提交了最新 Generated 代码 |
| CI 是否通过 | lint、generate、breaking 是否通过 |

## 总结

Protobuf 自动生成流程的核心在于统一协议定义、规范化工具链、自动化检查流程。推荐采用以下组合：

```text
Buf 管理 proto lint 与 breaking check
protoc / Buf 生成 Swift、Go、TypeScript 等代码
Makefile 提供统一本地命令
GitHub Actions 检查生成代码是否同步
生成代码随 SDK 或移动端项目入库
```

该方案兼顾工具链统一、多语言扩展、CI 自动校验、本地开发便利性与协议演进安全性。最终目标是让开发者只需要关注 `.proto` 协议定义本身，而不是手动维护不同语言的协议模型。

参考资料：

1. [Buf Documentation](https://buf.build/docs/)
2. [Protocol Buffers Style Guide](https://protobuf.dev/programming-guides/style/)
3. [gRPC Swift](https://github.com/grpc/grpc-swift)
4. [Swift Protobuf](https://github.com/apple/swift-protobuf)
