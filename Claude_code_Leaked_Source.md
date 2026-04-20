# Claude Code 源码泄露分析及逆向学习项目

## 📖 项目简介

2026年3月31日，Anthropic 官方命令行开发工具 **Claude Code CLI** 的完整 TypeScript 源代码因 npm 发布包中的 Source Map 配置失误而遭到泄露。本仓库旨在收集、整理并分析此次泄露的源代码资料，为开发者、安全研究者和 AI Agent 爱好者提供一份深入理解 Claude Code 内部架构、工具调用策略与安全设计的系统性参考。

无论你是希望复现 Claude Code 强大的编码代理能力、研究 Anthropic 的提示工程实践，还是单纯对现代 CLI 工具的设计模式感兴趣，这里都将成为你宝贵的学习材料。

> **特别说明**：本仓库由大学学生维护，仅用于**教育、防御性安全研究及软件供应链分析**，不主张对原始代码的任何所有权，亦非 Anthropic 官方资源。

---

## 🔓 泄露事件概述

此次泄露由安全研究员 [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现并公开披露。他在社交媒体上指出，Anthropic 发布至 npm 注册表的 Claude Code 包中包含了指向完整、未混淆 TypeScript 源代码的 Source Map 文件，而这些源文件恰好托管在 Anthropic 的 R2 存储桶中并可被公开下载。

> **"Claude code source code has been leaked via a map file in their npm registry!"**
> — [@Fried_rice, 2026年3月31日](https://x.com/Fried_rice/status/2038894956459290963)

此次事件使外界得以一窥 Anthropic 旗舰开发工具的完整内部构造，包括近 **1,900 个源文件、512,000 余行代码**，涵盖其工具系统、多代理协作、MCP 协议集成、安全分类器以及独特的四层决策流水线。

---

## 🛠️ 技术栈一览

| 类别               | 技术                                                                    |
| ------------------ | ----------------------------------------------------------------------- |
| **运行时**         | [Bun](https://bun.sh)                                                   |
| **开发语言**       | TypeScript (严格模式)                                                   |
| **终端 UI 框架**   | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| **CLI 解析**       | [Commander.js](https://github.com/tj/commander.js) (带额外类型定义)     |
| **模式校验**       | [Zod v4](https://zod.dev)                                               |
| **代码搜索**       | [ripgrep](https://github.com/BurntSushi/ripgrep) (通过 GrepTool 调用)   |
| **协议支持**       | MCP SDK, LSP                                                            |
| **API 客户端**     | [Anthropic SDK](https://docs.anthropic.com)                             |
| **遥测与可观测性** | OpenTelemetry + gRPC                                                    |
| **特性开关**       | GrowthBook                                                              |
| **认证**           | OAuth 2.0, JWT, macOS Keychain                                          |

---

## 📁 项目结构与规模

泄露的 `src/` 目录包含约 **1,900 个 TypeScript/TSX 文件**，总代码量超过 **512,000 行**。主要目录组织如下：

```text
src/
├── main.tsx                 # 应用主入口（Commander.js CLI 解析 + Ink 渲染）
├── commands.ts              # 命令注册中心
├── tools.ts                 # 工具注册中心
├── Tool.ts                  # 工具基础类型定义
├── QueryEngine.ts           # LLM 查询引擎（核心 API 调用与流式响应处理）
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本追踪
│
├── commands/                # 斜杠命令实现（约 50 个）
├── tools/                   # 智能体工具实现（约 40 个）
├── components/              # Ink UI 组件（约 140 个）
├── hooks/                   # React Hooks
├── services/                # 外部服务集成
├── screens/                 # 全屏界面（Doctor、REPL、Resume）
├── types/                   # TypeScript 类型定义
├── utils/                   # 通用工具函数
│
├── bridge/                  # IDE 集成桥接层（VS Code、JetBrains）
├── coordinator/             # 多智能体协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── keybindings/             # 快捷键绑定
├── vim/                     # Vim 模式支持
├── voice/                   # 语音输入
├── remote/                  # 远程会话管理
├── server/                  # 服务器模式
├── memdir/                  # 持久化记忆目录
├── tasks/                   # 任务管理
├── state/                   # 状态管理
├── migrations/              # 配置迁移
├── schemas/                 # 配置模式定义（Zod）
├── entrypoints/             # 初始化逻辑
├── ink/                     # Ink 渲染器封装
├── buddy/                   # 伙伴精灵（彩蛋）
├── native-ts/               # 原生 TypeScript 工具
├── outputStyles/            # 输出样式
├── query/                   # 查询流水线
└── upstreamproxy/           # 上游代理配置
```

---

## 🏗️ 核心架构解析

### 1. 工具系统 (`src/tools/`)

Claude Code 可调用的每一个工具均实现为独立模块，包含输入模式定义、权限模型与执行逻辑。主要工具如下：

| 工具名称                                 | 功能描述                              |
| ---------------------------------------- | ------------------------------------- |
| `BashTool`                               | Shell 命令执行                        |
| `FileReadTool`                           | 文件读取（支持图片、PDF、Jupyter 等） |
| `FileWriteTool`                          | 文件创建 / 覆盖                       |
| `FileEditTool`                           | 文件局部修改（字符串替换）            |
| `GlobTool`                               | 基于通配符的文件名搜索                |
| `GrepTool`                               | 基于 ripgrep 的内容搜索               |
| `WebFetchTool`                           | 获取 URL 内容                         |
| `WebSearchTool`                          | 网络搜索                              |
| `AgentTool`                              | 派生子智能体                          |
| `SkillTool`                              | 执行预定义技能                        |
| `MCPTool`                                | 调用 MCP 服务器工具                   |
| `LSPTool`                                | 语言服务器协议集成                    |
| `NotebookEditTool`                       | Jupyter Notebook 编辑                 |
| `TaskCreateTool` / `TaskUpdateTool`      | 任务创建与更新                        |
| `SendMessageTool`                        | 智能体间消息传递                      |
| `TeamCreateTool` / `TeamDeleteTool`      | 团队智能体管理                        |
| `EnterPlanModeTool` / `ExitPlanModeTool` | 计划模式切换                          |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git 工作树隔离                        |
| `ToolSearchTool`                         | 延迟工具发现                          |
| `CronCreateTool`                         | 定时触发器创建                        |
| `RemoteTriggerTool`                      | 远程触发器                            |
| `SleepTool`                              | 主动模式等待                          |
| `SyntheticOutputTool`                    | 结构化输出生成                        |

### 2. 命令系统 (`src/commands/`)

用户以 `/` 前缀调用的斜杠命令，涵盖了从版本控制到配置管理的完整工作流：

| 命令                 | 功能描述              |
| -------------------- | --------------------- |
| `/commit`            | 智能生成 Git 提交信息 |
| `/review`            | 代码审查              |
| `/compact`           | 上下文压缩            |
| `/mcp`               | MCP 服务器管理        |
| `/config`            | 配置管理              |
| `/doctor`            | 环境诊断              |
| `/login` / `/logout` | 账户认证              |
| `/memory`            | 持久化记忆管理        |
| `/skills`            | 技能管理              |
| `/tasks`             | 任务管理              |
| `/vim`               | Vim 模式切换          |
| `/diff`              | 变更查看              |
| `/cost`              | 用量费用查询          |
| `/theme`             | 主题切换              |
| `/context`           | 上下文可视化          |
| `/pr_comments`       | PR 评论查看           |
| `/resume`            | 恢复历史会话          |
| `/share`             | 会话分享              |
| `/desktop`           | 桌面应用移交          |
| `/mobile`            | 移动应用移交          |

### 3. 服务层 (`src/services/`)

服务层负责与外部系统交互，并提供核心业务逻辑支持：

| 服务模块                 | 功能描述                                 |
| ------------------------ | ---------------------------------------- |
| `api/`                   | Anthropic API 客户端、文件 API、引导程序 |
| `mcp/`                   | MCP 服务器连接与管理                     |
| `oauth/`                 | OAuth 2.0 认证流程                       |
| `lsp/`                   | 语言服务器协议管理器                     |
| `analytics/`             | 基于 GrowthBook 的特性开关与分析         |
| `plugins/`               | 插件加载器                               |
| `compact/`               | 对话上下文压缩                           |
| `policyLimits/`          | 组织策略限制                             |
| `remoteManagedSettings/` | 远程托管设置                             |
| `extractMemories/`       | 自动记忆提取                             |
| `tokenEstimation.ts`     | Token 数量估算                           |
| `teamMemorySync/`        | 团队记忆同步                             |

### 4. 桥接系统 (`src/bridge/`)

桥接系统为 IDE 扩展（VS Code、JetBrains）与 Claude Code CLI 之间提供双向通信层：

- `bridgeMain.ts` — 桥接主循环
- `bridgeMessaging.ts` — 消息协议定义
- `bridgePermissionCallbacks.ts` — 权限回调处理
- `replBridge.ts` — REPL 会话桥接
- `jwtUtils.ts` — 基于 JWT 的认证工具
- `sessionRunner.ts` — 会话执行管理

### 5. 权限系统 (`src/hooks/toolPermission/`)

每次工具调用前均会进行权限校验。根据配置的权限模式（如 `default`、`plan`、`bypassPermissions`、`auto` 等），系统或向用户发起批准/拒绝询问，或自动放行。

### 6. 特性开关 (Feature Flags)

利用 Bun 的 `bun:bundle` 特性标志实现死码消除，在构建时移除未启用的功能代码：

```typescript
import { feature } from "bun:bundle";

const voiceCommand = feature("VOICE_MODE")
  ? require("./commands/voice/index.js").default
  : null;
```

主要特性开关包括：`PROACTIVE`、`KAIROS`、`BRIDGE_MODE`、`DAEMON`、`VOICE_MODE`、`AGENT_TRIGGERS`、`MONITOR_TOOL`。

---

## 🔎 关键文件详解

### `QueryEngine.ts`（约 46,000 行）

LLM 查询的核心引擎。负责处理流式响应、工具调用循环、思考模式（thinking mode）、重试逻辑以及 Token 计数。它是连接 Claude Code 与 Anthropic API 的中枢。

### `Tool.ts`（约 29,000 行）

定义所有工具的基类、接口与类型，包括输入模式、权限模型及进度状态类型，是工具系统的基础抽象层。

### `commands.ts`（约 25,000 行）

管理所有斜杠命令的注册与执行。通过条件导入机制，可根据不同运行环境加载差异化的命令集合。

### `main.tsx`

基于 Commander.js 的 CLI 解析器与 React/Ink 渲染器的初始化入口。启动时通过并行预取 MDM 设置、钥匙串数据及 GrowthBook 初始化，显著缩短冷启动时间。

---

## 💡 值得注意的设计模式

### 并行预取 (Parallel Prefetch)

在模块解析之前，并行启动 MDM 设置读取、钥匙串预取及 API 预连接，以提升启动速度：

```typescript
// main.tsx 中作为副作用提前执行
startMdmRawRead();
startKeychainPrefetch();
```

### 懒加载 (Lazy Loading)

OpenTelemetry（约 400KB）、gRPC（约 700KB）等重型模块通过动态 `import()` 延迟加载，仅在需要时引入。

### 智能体集群 (Agent Swarms)

通过 `AgentTool` 派生子智能体，并由 `coordinator/` 协调多智能体并行工作。`TeamCreateTool` 则支持团队级别的并行任务分发。

### 技能系统 (Skill System)

可复用的工作流定义在 `skills/` 目录下，并通过 `SkillTool` 执行。用户可自由添加自定义技能。

### 插件架构 (Plugin Architecture)

内置及第三方插件均通过 `plugins/` 子系统加载，提供了良好的扩展性。

---

## 🔌 MCP 服务器使用指南

本仓库同时提供了一个 MCP 服务器，允许你在任何 Claude 会话中直接探索 Claude Code 的源代码结构。

### 快速开始

运行以下单条命令即可完成克隆、构建与注册：

```bash
git clone https://github.com/Atharvsinh-codez/claude-code.git ~/claude-code \
  && cd ~/claude-code/mcp-server \
  && npm install && npm run build \
  && claude mcp add claude-code-explorer -- node ~/claude-code/mcp-server/dist/index.js
```

### 可用工具

添加成功后，Claude 将获得以下源代码探索工具：

| 工具名称             | 功能描述                                 |
| -------------------- | ---------------------------------------- |
| `list_tools`         | 列出所有约 40 个智能体工具及其源文件路径 |
| `list_commands`      | 列出所有约 50 个斜杠命令及其源文件路径   |
| `get_tool_source`    | 读取指定工具的完整源代码                 |
| `get_command_source` | 读取指定命令的完整源代码                 |
| `read_source_file`   | 按路径读取 `src/` 中的任意文件           |
| `search_source`      | 在整个源代码树中进行内容搜索             |
| `list_directory`     | 浏览 `src/` 目录结构                     |
| `get_architecture`   | 获取高级架构概览                         |

此外还提供了一系列**提示词（Prompts）**用于引导探索：
- `explain_tool` — 深入解析特定工具的工作原理
- `explain_command` — 理解特定命令的实现细节
- `architecture_overview` — 架构全景导览
- `how_does_it_work` — 解释权限、MCP、桥接等子系统
- `compare_tools` — 并排比较两个工具

### 使用示例

```text
> "BashTool 是如何工作的？"
> "搜索权限检查的相关代码。"
> "显示 /review 命令的源代码。"
> "解释 MCP 客户端集成的实现方式。"
```

---

## ⚖️ 免责声明与版权说明

- 本仓库归档的源代码源自 **2026 年 3 月 31 日** Anthropic npm 注册表意外暴露的快照。
- 所有原始代码的版权与知识产权归 **Anthropic, PBC.** 所有。
- 本项目由大学学生维护，**仅用于教育、防御性安全研究及软件供应链分析**。
- 本项目**与 Anthropic 无任何关联、未获其认可，亦非其官方资源**。
- 若您认为本仓库内容侵犯了您的权益，请通过 GitHub 渠道联系我们。

---

> **最后更新**：2026 年 3 月 31 日