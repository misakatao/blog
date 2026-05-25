---
title: "Claude Code 与 GitHub 协作：从辅助编码到工程化落地"
description: "在 GitHub 工作流中引入 Claude Code 的完整实践指南，涵盖 Issue 驱动、分支策略、Commit 规范、PR Review、权限控制与团队落地路径"
date: 2026-05-25T14:00:00+08:00
slug: claude-code-github-collaboration
image: "cover.svg"
categories:
    - 技术
tags:
    - Claude Code
    - GitHub
    - AI
    - 工程化
    - DevOps
---

AI 编程工具正在改变开发者的日常工作方式，但"能用"和"用好"之间有很大距离。这篇文章整理了在 GitHub 项目中使用 Claude Code 进行协作开发的实践经验，核心目标不是让 AI 替代开发者，而是让它成为一个可控、可追踪、可审查的工程协作者。

<!--more-->

## 核心原则

在展开具体实践之前，先明确几条底线：

- Claude Code 负责辅助分析、生成、重构、解释和补充测试
- 开发者负责需求判断、架构决策、代码审查和最终合并
- 所有 AI 生成代码必须经过测试、Review 和必要的人工验证
- 所有关键变更都应通过 Issue → Branch → Commit → PR 留痕

这不是对 AI 能力的不信任，而是工程纪律。任何进入主干的代码，无论来源，都应该经过相同的质量门禁。

## 适用场景与边界

### 适合交给 Claude Code 的任务

| 场景 | 典型任务 |
|------|----------|
| 需求澄清 | 根据 Issue 拆解任务、分析代码结构、输出方案和风险点 |
| 代码实现 | 中小规模功能、工具函数、接口适配层、局部逻辑修改 |
| 测试补充 | 生成测试用例、覆盖边界条件、根据 Bug 复现步骤生成失败测试 |
| PR Review | 总结变更、检查潜在 Bug、发现未覆盖的测试场景 |
| 文档维护 | 更新 README、CHANGELOG、API 文档，整理技术记录 |

### 不应完全交给 AI 的任务

- 涉及生产密钥、支付、账户权限、用户隐私数据的修改
- 大规模架构迁移（缺少明确设计文档时）
- 数据库破坏性变更：删除字段、重建表、批量清洗生产数据
- 安全敏感逻辑：鉴权、加密、密钥管理、权限控制
- 无测试覆盖、无 Review、无回滚方案的核心业务改动

对于高风险任务，可以让 Claude Code 参与分析和生成草案，但决策和合并必须由人完成。

## GitHub 协作模型

推荐的完整协作链路：

```text
Issue → 方案讨论 → 创建分支 → Claude Code 辅助实现 → 本地验证
    → Pull Request → Claude Code Review → 人工 Review → CI 通过 → 合并
```

### Issue 驱动开发

每个任务应先创建 GitHub Issue，避免在模糊上下文中直接让 AI 改代码。一个好的 Issue 应该包含：

- **背景**：为什么要做这个任务
- **目标**：明确的交付物清单
- **非目标**：本次不做什么
- **涉及范围**：模块、文件、API
- **验收标准**：功能、测试、兼容性
- **风险点**：已知的技术约束或依赖

### 分支策略

Claude Code 参与开发时，始终使用独立分支：

```text
feature/<issue-id>-short-description
fix/<issue-id>-short-description
chore/<issue-id>-short-description
docs/<issue-id>-short-description
```

不要让 Claude Code 直接在 `main`、`master`、`release` 分支上提交代码。

## Claude Code 使用规范

### 先读后写

不要直接说"帮我实现这个功能"。正确的做法是先让 Claude Code 理解上下文：

```text
请先阅读以下文件并总结当前实现：
- Sources/App/FeatureA.swift
- Sources/Core/Service.swift
- Tests/CoreTests/ServiceTests.swift

然后说明：
1. 当前代码结构
2. 需要修改的位置
3. 可能影响的测试
4. 实现计划

在我确认前，不要修改代码。
```

### 先计划后执行

复杂任务必须分两步走。第一步输出计划：

```text
请基于当前仓库分析这个 Issue 的实现方案。只输出计划，不要修改文件。
请包含：修改文件、新增类型或函数、兼容性影响、测试计划、风险点。
```

开发者确认后，第二步才让 Claude Code 动手修改。

### 限制修改范围

每次任务尽量约束 Claude Code 的改动边界：

```text
只允许修改以下文件：
- Sources/BLEKit/BLECentral.swift
- Sources/BLEKit/PairingSession.swift
- Tests/BLEKitTests/PairingSessionTests.swift

不要修改 Package.swift，不要调整公共 API，除非先说明原因。
```

### 要求输出变更摘要

每次改完代码后，要求 Claude Code 输出结构化总结：文件级变更列表、关键实现逻辑、新增或修改的测试、已知风险、建议 Reviewer 重点检查的位置。

## Git Commit 规范

### Commit 粒度

AI 生成的代码也应遵守正常 Git 规范：

- 一个 Commit 只解决一个明确问题
- 不要把格式化、重构、功能开发混在一个 Commit 中
- 大任务拆成多个可 Review 的 Commit
- 每个 Commit 都应能说明"为什么改"

### Commit Message

使用 Conventional Commits：

```text
feat: add BLE pairing session
fix: handle missing vehicle public key
docs: update GitHub collaboration guide
test: add command encoding tests
refactor: split BLE transport from command protocol
chore: update CI workflow
```

### AI 协作者署名

如果团队允许标记 AI 协作，可在 Commit 或 PR 描述中声明：

```text
Assisted-by: Claude Code
```

或者在 PR 描述中写明 AI 参与了哪些环节（初始实现草案、单元测试补充、PR 变更摘要等），核心设计与最终 Review 由人工完成。

是否保留 AI 署名应根据团队规范决定。

## Pull Request 实践

### PR 尺寸控制

| 规模 | 行数 | 建议 |
|------|------|------|
| 小型 | 50–300 行 | 适合快速 Review |
| 中型 | 300–800 行 | 需要清晰说明 |
| 大型 | 800+ 行 | 应拆分，除非是自动生成代码 |

Claude Code 更适合处理边界清晰的小型和中型 PR。

### PR 描述结构

一个完整的 PR 描述应包含：背景、变更内容清单、测试情况、风险说明、AI 使用说明、Reviewer 重点关注位置。

### 用 Claude Code 做 Review

```text
@claude 请 Review 这个 PR。重点关注：
1. 是否存在边界条件遗漏
2. 是否有并发或状态管理问题
3. 是否影响现有公共 API
4. 测试覆盖是否足够
5. 是否有安全或隐私风险

请不要直接修改代码，只输出 Review 意见。
```

## 安全与权限控制

### Secrets 管理

不要把以下内容暴露给 Claude Code：API Key、私钥、数据库密码、OAuth Client Secret、生产环境配置、用户隐私数据。

如果需要理解配置结构，提供脱敏样例：

```env
API_BASE_URL=https://example.com
API_KEY=<REDACTED>
DATABASE_URL=<REDACTED>
```

### Prompt Injection 防护

Claude Code 会读取仓库中的文件、Issue、PR 评论和上下文，需要警惕恶意提示注入。防护要点：

- 不把敏感密钥暴露给运行环境
- 不让 Claude Code 对外发送敏感内容
- 对来自 Issue、PR 评论、外部文档的指令保持不信任
- 对外部贡献者 PR 使用更严格的权限

### 本地执行安全

在本地使用 Claude Code 时，执行命令前检查：是否会删除文件、是否会修改 Git 历史、是否会安装未知依赖、是否会访问网络或上传数据、是否会读取密钥文件。高风险命令必须人工确认。

## GitHub Actions 集成

### 触发方式

建议优先使用显式触发（`@claude`），不建议对所有 Issue 和 PR 自动触发，避免成本失控和误操作。

### 适合自动化的任务

- PR 摘要生成
- Issue 分类
- 测试失败分析
- 文档更新建议

### 不建议完全自动化的任务

- 自动合并 PR
- 自动修改生产配置
- 自动处理密钥和权限
- 自动执行数据库迁移
- 自动发布生产版本

## 团队落地路径

建议分三阶段引入：

### 阶段一：只读辅助

代码解释、PR 摘要、Issue 分析、CI 日志分析。目标是建立信任，降低误用风险。

### 阶段二：受控修改

小功能实现、单元测试补充、文档更新、局部重构。目标是提升效率，同时保留人工控制。

### 阶段三：流程自动化

GitHub Actions 集成、自动 PR 摘要、自动测试建议、自动文档维护建议。目标是把重复协作环节产品化，但不跳过人工 Review。

## 检查清单

开发前：Issue 描述清晰、验收标准明确、已创建独立分支、已让 Claude Code 先分析再执行。

开发中：修改范围受控、没有暴露 Secrets、没有引入不必要依赖、复杂逻辑已要求解释。

提交前：编译通过、测试通过、Lint 通过、Commit 粒度合理、Message 符合规范。

合并前：CI 全部通过、至少一名人工 Reviewer 通过、高风险变更已二次确认、有回滚方案。

## 总结

Claude Code 与 GitHub 的最佳协作方式，不是"让 AI 自动写完并合并代码"，而是建立一个可控、可审查、可回滚的工程流程。

关键实践：先 Issue 后开发、先计划后修改、小步提交清晰 PR、最小权限保护 Secrets、AI 辅助人工负责、所有变更必须经过测试和 Review。

当团队把 Claude Code 放进标准 GitHub 流程中，而不是绕过流程时，它才能真正提升研发效率并降低协作风险。
