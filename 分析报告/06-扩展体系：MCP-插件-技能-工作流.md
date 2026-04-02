# 06. 扩展体系：MCP、插件、技能、工作流

## 平台化是 Claude Code 的长期方向

从 [src/services/mcp/client.ts](../src/services/mcp/client.ts)、[src/skills/loadSkillsDir.ts](../src/skills/loadSkillsDir.ts)、[src/utils/plugins/loadPluginCommands.ts](../src/utils/plugins/loadPluginCommands.ts)、[src/commands.ts](../src/commands.ts)、[src/tools.ts](../src/tools.ts) 可以看出，Claude Code 的长期演进方向并不是不断往主仓库塞新功能，而是把系统建设成一个可扩展的 agent 平台。

这套扩展体系至少包含四种能力来源：

1. 内建命令和工具
2. skills
3. plugins
4. MCP servers

它们共同构成了 Claude Code 的“能力装配面”。

## MCP：最重要的外部能力协议

### MCP 在系统中的角色

MCP 不是附属接口，而是 Claude Code 与外部世界对接的标准扩展协议。它可以导入：

- tools
- resources
- prompts
- auth flows
- elicitation flows

从架构角度看，MCP 解决的是“如何把外部能力纳入 agent 内核而不把每个外部系统做成内建代码”。

### MCP 客户端设计的几个关键点

[src/services/mcp/client.ts](../src/services/mcp/client.ts) 展示了非常成熟的设计：

- 支持 stdio、SSE、streamable HTTP、WebSocket 等多种 transport
- 有 session expired 与 auth error 的专门错误模型
- 有 auth cache 和 refresh 机制
- 会将外部 server capability 规范化为本地 Tool/Resource/Command 形态
- 支持结果持久化、内容截断、大输出处理、图像降采样等现实问题

这说明 MCP 集成不是“SDK wrapper”，而是一个完整的 external capability adapter layer。

### Rust 重构启示

Rust 系统应把 MCP 看成一级子系统，而不是外部插件。建议单独做：

- transport abstraction
- capability normalization
- auth/session manager
- resource and prompt importer
- tool result safety layer

## 技能系统：面向 prompt 的轻量扩展

[src/skills/loadSkillsDir.ts](../src/skills/loadSkillsDir.ts) 说明 skills 的本质是“通过 Markdown + frontmatter 定义的 prompt 级扩展单元”。

它支持的元信息包括：

- name / description / whenToUse
- arguments
- allowed-tools
- model / effort
- execution context
- hooks
- paths 约束

### 这意味着什么

skills 不是传统插件，也不是普通文档，而是一种介于“命令模板”和“知识驱动能力声明”之间的扩展对象。

它的工程价值在于：

- 扩展成本低
- 可被版本化和仓库化
- 能被系统理解为命令/提示资产
- 对非编译型扩展特别友好

这是新一代 Code Agent 很值得保留的设计，因为它为“领域知识封装”提供了极低门槛的宿主形态。

## 插件系统：命令与技能的打包扩展

[src/utils/plugins/loadPluginCommands.ts](../src/utils/plugins/loadPluginCommands.ts) 展示了插件系统的设计原则：

- 插件通过 Markdown 和 frontmatter 定义命令或技能
- 可以进行变量替换、用户配置替换、参数替换
- 能从插件目录递归收集 markdown 文件
- skill file 与普通 command file 可以统一转换为 Command

### 关键意义

Claude Code 并没有把插件系统做成“运行任意 JS 的黑盒插件”，而是优先把插件做成：

- 可解析
- 可限制
- 可声明
- 可审计

的命令/技能扩展源。

这比直接暴露完整脚本 API 更容易与权限和可观测性体系结合。

## 命令、技能、插件并非三套孤立体系

从 [src/commands.ts](../src/commands.ts) 的注册流程可以看出：

- 内建命令
- 动态技能命令
- 插件命令

最终都会汇入统一的 Command 视图。

这说明 Claude Code 在架构上追求的是：

> 多种来源的能力定义，统一收敛到少数几个运行时抽象对象上。

这点非常值得在 Rust 重构中保留。否则扩展体系很容易碎裂成多个互不兼容的子系统。

## 工作流脚本是面向自动化组合的扩展

从 `WORKFLOW_SCRIPTS` feature 与 `WorkflowTool` 的装配方式可见，系统除了单个工具和单个命令外，也在探索更高层级的工作流封装。

这说明扩展体系分三层：

1. 原子能力层：tools
2. 语义封装层：skills/plugins/commands
3. 编排组合层：workflows

未来 Rust 版若只保留第一层和第二层，而忽略第三层，会限制系统往企业自动化平台演进。

## 扩展体系的核心设计原则

### 原则一：能力来源多样，但运行时抽象统一

无论能力来自内建、MCP、插件还是技能，最终都应映射为少量可调度的运行时对象。

### 原则二：扩展元数据必须可解析、可静态分析

frontmatter、schema、tool metadata、resource metadata 都说明系统非常重视扩展定义的结构化可读性。

### 原则三：扩展应纳入权限与策略框架

MCP auth、plugin restrictions、allowed-tools、source policy 等逻辑说明：扩展不应绕过治理层。

### 原则四：协议层要与宿主层解耦

MCP transport 多态说明了这一点。协议与宿主解耦，才能跨 CLI、SDK、远程模式复用。

## Rust 重构建议

### 建议模块划分

1. `ext-core`
2. `ext-mcp`
3. `ext-skill`
4. `ext-plugin`
5. `ext-workflow`
6. `ext-policy`

### 建议统一抽象

- `CapabilitySource`
- `CapabilityDescriptor`
- `NormalizedTool`
- `NormalizedCommand`
- `ExtensionPolicy`
- `TransportAdapter`

### 实现优先级建议

1. 先完成 MCP tool/resource 接入
2. 再完成 skill frontmatter 解析
3. 再做插件目录与用户配置系统
4. 最后做 workflow 编排层

## 本章结论

Claude Code 的扩展体系说明，它已经从“功能仓库”演进为“agent 平台”。新一代 Rust 项目若想真正超越，而不是只做一个更快的 CLI，就必须在设计上把扩展体系作为平台核心，而不是后期兼容层。

## 代码证据

- [src/services/mcp/client.ts](../src/services/mcp/client.ts)
- [src/skills/loadSkillsDir.ts](../src/skills/loadSkillsDir.ts)
- [src/utils/plugins/loadPluginCommands.ts](../src/utils/plugins/loadPluginCommands.ts)
- [src/commands.ts](../src/commands.ts)
- [src/tools.ts](../src/tools.ts)
- [src/plugins/bundled/index.ts](../src/plugins/bundled/index.ts)
- [docs/05-hidden-commands.md](../docs/05-hidden-commands.md)
