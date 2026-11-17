# 2026 全网最新演进与生态全景速查（v2.1.88 → v2.1.326）

> **更新基线**：2026 年 11 月（Claude Code `v2.1.326`、Claude 5.5 家族与 Fable 5.2、MCP 规范 `2026-07-28`、`@anthropic-ai/sdk` `v1.9.0`、`@anthropic-ai/claude-agent-sdk` `v0.3.326`）。
>
> **阅读指南**：本课程正文与源码行号以 **Claude Code v2.1.88** 快照为精确基准（确保所有文件路径、行号与核心数字 1:1 可复现）。从 2026 年 3 月底的 `v2.1.88` 到 2026 年 11 月的 `v2.1.326`，Anthropic 发布了近 240 个小版本，AI 编程代理行业也经历了模型代际跃迁、MCP 无状态重构与 Linux 基金会标准化浪潮。本文档为你提供从 `v2.1.88` 对照到全网最新进展的完整工程地图。

---

## 一、Claude Code 核心演进（v2.1.88 → v2.1.326）

从 `v2.1.88` 到 `v2.1.326`，Claude Code 的底层 `AsyncGenerator` 主循环、`Tool` 抽象契约与多层上下文体系完全延续了本课程剖析的架构，同时在执行引擎、权限默认策略、多代理编排、云端调度和插件形态上完成了六大跨越：

### 1.1 Code Mode：批量 JavaScript 工具编排（v2.1.318–v2.1.326，对应 Ch04/Ch06/Ch16）

传统的 `tool_use` 循环每次工具调用与串联都需要一次完整的 LLM 往返，且大体积 MCP 返回结果会直接涌入上下文窗口。`v2.1.323` 正式文档化的实验特性 **Code Mode**（`CLAUDE_CODE_EXPERIMENTAL_CODE_MODE=1`）引入了更高阶的执行范式：

- **工作原理**：Claude 不再逐个返回离散的 `tool_use` 块，而是直接编写一段 JavaScript 脚本，在受控的 Bun REPL 环境中批量执行工具调用——可在脚本内并发调用多个工具、将上游工具输出直接通过代码逻辑管道传给下游工具、并在本地过滤和聚合海量 MCP 载荷后，只把精简摘要返回给上下文窗口。在重度 MCP 任务中可节省 **40%–77%** 的上下文 Token。
- **安全与可观测**：执行前，Auto Mode 分类器会对整段生成脚本做静态与语义安全审查（拦截 `\uXXXX` 混淆逃逸或 Hook 篡改不匹配）；终端中可通过 `Ctrl+O` 查看生成的脚本源码。
- **多代理协同（v2.1.326）**：开启 **Agent Teams** 时已原生兼容 Code Mode，进程内（in-process）队友代理同样支持批量脚本化工具调用。

### 1.2 权限默认模式分流与沙箱收紧（v2.1.283–v2.1.325，对应 Ch10/Ch11）

- **交互式会话默认开启 Auto Mode（v2.1.283+）**：终端与 VS Code 交互式会话不再默认停留在每次写操作都弹窗的传统手动模式，而是默认进入由后台安全分类器护航的 **Auto Mode**（可通过 `/auto-mode-setup` 或 `permissions.defaultMode` 配置，且在 `v2.1.158+` 已扩展支持 AWS Bedrock、Google Vertex AI 与 Microsoft Foundry）。
- **无头模式与 SDK 默认保持 Manual Mode（v2.1.302+）**：`claude -p` 与 Claude Agent SDK 运行默认回退为严格的 Manual（`default`）模式，除非显式传入 `--permission-mode auto` 或白名单规则，防止无人值守流水线意外放权。
- **沙箱与 Hook 策略加固（v2.1.300–v2.1.325）**：
  - `v2.1.300`：对 `PreToolUse` 与 `PermissionRequest` Hook 改写后的 `updatedInput` 强制执行目标工具的 Zod Schema 校验。
  - `v2.1.323`：为 SDK / `-p` 模式下的长耗时 Hook 增加 30 秒心跳事件（`CLAUDE_CODE_HOOK_HEARTBEAT_INTERVAL_MS`）。
  - `v2.1.325`：收紧沙箱权限，默认不再因开启沙箱就自动放行所有 Bash 命令（调整 `autoAllowBashIfSandboxed` 默认行为），并封堵对主目录敏感路径与 `.credentials.json` 的写入；同时规范策略 Hook 退出码语义——退出码 `1` 且无 JSON 决策时不再阻断执行，必须返回退出码 `2` 或显式输出 `"decision": "block"`。

### 1.3 Agent Teams、递归子代理与分级 Effort（v2.1.166–v2.1.326，对应 Ch16/Ch17）

- **递归子代理（v2.1.166–v2.1.176）**：子代理（Subagents）支持继续派生下一级子代理，后台嵌套深度上限为 **5 层**。
- **默认后台异步运行（v2.1.198）**：子代理默认以异步后台方式启动，不再阻塞主会话输入，子代理触发的权限确认会自动冒泡到主会话界面。
- **细粒度思考预算（Effort & Ultracode）**：
  - `v2.1.284` 起，Effort 等级按模型独立持久化（`modelSettings`），并将 `/effort ultracode` 拆分为独立开关。
  - `v2.1.31x` 新增环境变量 `CLAUDE_CODE_SUBAGENT_EFFORT`（独立控制后台辅助子代理的 Token 消耗强度）以及 `xhigh` 档位的 `ultrathink`。

### 1.4 Cloud Scheduled Tasks（原 Routines）与 Channels（对应 Ch12/Ch19）

- **Routines 正式更名为 Cloud Scheduled Tasks（2026 年 10 月）**：运行在 Anthropic 托管云沙箱（或企业自托管环境）中的无人值守代理任务，CLI 命令 `/routines`、`/schedule` 与触发端点 `POST .../fire` 保持兼容。支持三类触发器：
  1. **Cron 定时调度**（最小粒度 1 小时，UTC）
  2. **API Webhook 触发**（独立 Bearer Token 鉴权）
  3. **GitHub 事件触发**（如 `pull_request.opened`、Issue、Release）
- **Channels 双向事件推送成熟化**：通过 `claude --channels plugin:telegram@claude-plugins-official`（支持 Telegram、Discord、iMessage、Slack 及 CI/Sentry Webhook），外部 MCP Server 可将实时告警或手机端指令**主动推送**进本机正在运行的 Claude Code 会话，直接复用本机文件系统与 Git 上下文。
- **Remote Control 实时流式同步（v2.1.321–v2.1.326）**：`claude remote-control` 新增 `--settings <file>` 支持，并实现了向手机 App 与 Web 浏览器的**逐 Token 实时流式推送**；同时 **Claude in Chrome** 正式 GA（`v2.1.198`），**Claude Desktop Linux 版**开放公测。

### 1.5 Plugins 生态：JavaScript "Mods"、CI 评测与企业治理（对应 Ch14）

- **JavaScript "Mods"（v2.1.300+）**：在声明式 `.claude-plugin/plugin.json` 基础上，插件现已支持编写 JavaScript UI/生命周期扩展模块（Mods）——可绘制自定义终端面板、在输入框渲染 `virtualText` 装饰提示、并通过 `$.ui.view({ agentId })` 动态切换多代理转录视图。
- **`claude plugin eval` 自动化评测**：支持在 CI 中对插件执行对照评测（分别在启用/禁用插件的隔离会话中运行基准任务并自动打分，低于阈值返回非零退出码）。
- **企业级安全锁定（v2.1.292–v2.1.323）**：新增 `strictKnownMarketplaces`、`extraKnownMarketplaces`、`disableSideloadFlags`、插件级 OpenTelemetry 日志，以及 `v2.1.323` 的**插件版本/Git Tag 强制锁定（Pinning）**。

### 1.6 工作区指令兼容：`AGENTS.md` 原生回退加载（v2.1.277，对应 Ch07）

- 在 `v2.1.88` 时，Claude Code 仅在 `/init` 初始化扫描时吸收 `AGENTS.md`；自 **`v2.1.277`（2026 年 9 月）** 起，当仓库不存在 `CLAUDE.md` 时，Claude Code 会在运行时**自动回退读取 `AGENTS.md`**，实现了与开放标准的零配置兼容。
- 此外，`v2.1.166+` 新增 `/cd` 斜杠命令，支持在会话中途切换工作目录而**不使已建立的 Prompt Cache 失效**。

---

## 二、Anthropic 模型矩阵与 SDK 升级（2026.11）

### 2.1 Claude 5.5 家族与全新 Fable / Mythos 层级

2026 年下半年，Anthropic 完成了从 Claude 4.6/4.7/4.8 向 **Claude 5 / 5.5** 世代的切换，并在传统三档（Opus / Sonnet / Haiku）之上开辟了面向极限工程与科研推理的第四档旗舰——**Claude Fable**（及其受限科研/网安版本 **Claude Mythos**）。全系主力模型均原生标配 **1M Token 上下文窗口**、**128K 最大输出 Token** 与 **Adaptive Thinking（自适应思考）**：

| 模型 | 发布时间 | API Model ID | 标准定价（输入 / 输出，每百万 Token） | 核心定位 |
|------|---------|-------------|-------------------------------------|---------|
| **Claude Fable 5.2** *(含 Mythos 5.2)* | 2026-11-09 | `claude-fable-5-2` | \$8.00 / \$40.00（Fast Mode: \$16 / \$80） | 极限软件工程、前沿数学与科学推理、数小时级超长程自治任务 |
| **Claude Opus 5.5** | 2026-09-22 | `claude-opus-5-5` | \$4.00 / \$20.00 | 复杂代码库重构、长程多代理主力模型（Claude Code Pro/Team 默认模型） |
| **Claude Sonnet 5.5** | 2026-09-28 | `claude-sonnet-5-5` | \$2.00 / \$10.00（Cache Read \$0.10） | 日常生产研发、企业级 Agent 工作流、高性价比均衡首选 |
| **Claude Haiku 5.5** | 2026-10-07 | `claude-haiku-5-5` | \$0.10 / \$0.50 起 | 极低延迟路由、高频分类提取、大规模并行轻量 Subagent |

> **退役提醒**：Claude Code `v2.1.316` 已记录将于 **2026 年 11 月 30 日** 正式下线 `claude-sonnet-4-5-20250929` 与 Opus 4.8 Fast Mode（平滑迁移至 `claude-sonnet-5-5` 与 `claude-opus-5-5`）。本课程实战章节（Ch03、Ch09、Ch20、Ch22）的独立可运行代码已同步升级为 `claude-sonnet-5-5` 与 `claude-haiku-5-5`。

### 2.2 `@anthropic-ai/sdk` v1.9.0（API `2026-10-21`）重大变更

2026 年 10 月 21 日，Anthropic 官方 TypeScript SDK 发布 `1.0.0` 大版本（截至 2026 年 11 月中旬最新为 **`1.9.0`**，协议改用 Apache-2.0，要求 Node.js 20 LTS+ 与 TypeScript >= 5.0），默认请求头 `anthropic-version` 从沿用三年的 `2023-06-01` 升级为 **`2026-10-21`**：

1. **移除采样温度参数（`temperature` / `top_p` / `top_k`）**：TypeScript 类型层面直接移除，API 运行时传入任何值（哪怕是 `temperature: 1.0`）都会返回 `HTTP 400`——推理模型时代全面改由模型自适应采样与 `effort` / `thinking` 控制。
2. **`MessageParam` 拆分为角色联合类型**：按 `user` / `assistant` / `system` 拆分为可辨识联合类型，自定义接口不可再 `interface X extends MessageParam`（需改用交叉类型 `&`），跨消息操作 `content` 时需先收窄 `role`。
3. **认证头切换为 `Authorization: Bearer <key>`**：取代早期的 `X-Api-Key` 默认头（服务端仍双向兼容，但自定义网关与测试 Mock 需留意）。
4. **服务端压缩取代客户端 Tool Runner 压缩**：移除客户端 `compactionControl`，全面转向服务端原生 Context Compaction（触发阈值 >= 50,000 tokens）。

### 2.3 Claude Agent SDK 独立分包与 CLI 对齐（对应 Ch18）

Anthropic 已将早期的 Claude Code SDK 正式更名并拆分为独立包，版本号与 Claude Code CLI 保持严格同步：

- **TypeScript**：`@anthropic-ai/claude-agent-sdk`（最新 **`v0.3.326`**，对齐 `@anthropic-ai/claude-code` `v2.1.326`；`@anthropic-ai/claude-code` 现仅作为纯 CLI 包发布）。
- **Python**：`claude-agent-sdk`（最新 **`v0.2.183`**，PyPI 安装，模块导入名为 `claude_agent_sdk`，配置类为 `ClaudeAgentOptions`，内置打包 CLI 二进制）。

---

## 三、MCP 协议无状态重构与开放标准基金会（AAIF）

### 3.1 MCP 规范五代演进（从 `2024-11-05` 到 `2026-07-28`）

本课程 Ch12 与 Ch21 剖析的源码对应有状态会话时代的 MCP 协议（`2025-03-26` 至 `2025-11-25`）。而在 **2026 年 7 月 28 日**，MCP 发布了里程碑式的 **`2026-07-28` 规范**，完成了从“有状态双向会话协议”向“云原生无状态请求/响应协议”的架构重构：

| 规范版本 | 核心架构演进 |
|---------|-------------|
| **`2024-11-05`** | 初版规范：`stdio` 与旧版 `HTTP+SSE` 双通道传输，基础 Tools / Resources / Prompts 原语 |
| **`2025-03-26`** | 引入统一 **Streamable HTTP** 传输、OAuth 2.1 授权框架、**Tool Annotations**（`readOnlyHint` 等） |
| **`2025-06-18`** | 将 MCP Server 定义为 OAuth 2.0 Resource Server（RFC 9728 / RFC 8707）、新增 **Elicitation**（`elicitation/create`）与 **Structured Tool Output**（`outputSchema`）、强制要求 `MCP-Protocol-Version` 请求头、移除 JSON-RPC batching |
| **`2025-11-25`** | 新增 URL 模式 Elicitation、Sampling 工具调用能力、OAuth Client ID Metadata Documents（CIMD）、实验性 **Tasks** 原语（SEP-1686）、工具/资源图标元数据 |
| **`2026-07-28`（最新）** | **无状态协议核心重构（Stateless Core）**：<br>1. 移除强制 `initialize` 握手与 `Mcp-Session-Id` 会话状态，每个请求在 `_meta` 中自描述协议版本与客户端能力，按需通过 `server/discover` 探测服务端能力；<br>2. **多轮请求（MRTR）**：以返回 `input_required` 状态码并由客户端携带补充答案重试的方式，取代依赖持久长连接的反向 Sampling/Elicitation；<br>3. **网关友好 HTTP 头**：新增 `Mcp-Method` 与 `Mcp-Name` 请求头，使 API 网关与负载均衡器无需解析 JSON 包体即可路由与鉴权；<br>4. **缓存与可观测**：`tools/list` 支持 `ttlMs` / `cacheScope` 缓存提示，`_meta` 原生承载 OpenTelemetry Trace Context；<br>5. **正式扩展框架（Extensions）**：将长耗时 **MCP Tasks**、**MCP Apps** 与企业托管授权抽离为标准扩展模块，并确立 12 个月正式弃用周期。 |

与之配套，**MCP TypeScript SDK** 也形成了双轨维护格局：
- **v2 拆分包（面向 `2026-07-28` 无状态规范）**：`@modelcontextprotocol/server`、`@modelcontextprotocol/client`、`@modelcontextprotocol/core`（最新均为 **`2.4.0`**）。
- **v1 单体包（面向 `2025-11-25` 及有状态兼容，本课程 Ch21 使用）**：`@modelcontextprotocol/sdk`（最新维护版 **`1.32.2`**）。

同时，由 Anthropic、GitHub、Microsoft 等共建的 **Official MCP Registry** 已收录超过 **48,600 个活跃公开 MCP Server**（全网多注册表聚合总量突破 11.4 万个）。

### 3.2 Linux 基金会 Agentic AI Foundation (AAIF) 与 ACP 协议

2026 年是 AI 编程代理协议走向跨厂商中立治理的关键一年：

1. **Agentic AI Foundation (AAIF，https://aaif.io)**：隶属 Linux 基金会，截至 2026 年 10 月已有超过 305 家成员机构，托管 **10 大核心开源标准项目**：
   - **`MCP`**（Model Context Protocol）：工具与上下文连接标准
   - **`AGENTS.md`**（https://agents.md）：仓库级代理说明文件开放规范，已被 **60,000+** 开源项目采用
   - **`Agent Skills`**（https://agentskills.io）：由 Anthropic 于 **2026 年 10 月 22 日** 正式以 Apache 2.0 协议捐赠给 AAIF，基于 `SKILL.md` 三阶段渐进式披露（Discovery → Activation → Execution），已被 Claude Code、OpenAI Codex、GitHub Copilot、Cursor、Gemini、OpenCode 全面支持
   - **`A2A`**（Agent2Agent Protocol）：跨系统异构代理协作协议
   - **`Open Responses`**（OpenAI 贡献）、**`OpenEnv`**（Hugging Face 贡献）、**`OpenSandbox`**（Alibaba 贡献）、**`agentgateway`**、**`Agent Router`**、**`goose`**
2. **Agent Client Protocol (ACP)**：由 Zed 与 JetBrains 联合共建的“AI Agent 领域的 LSP 协议”（基于 `stdio` 的 JSON-RPC 2.0，提供 `@agentclientprotocol/sdk` 等多语言 SDK）。编辑器或编排平台（如 Zed、JetBrains 全家桶、OpenHands Agent Canvas）可通过 ACP 将 Claude Code、OpenAI Codex、Gemini CLI、OpenCode 作为标准化子进程挂载，统一接管 Diff 审查与文件/终端权限门控。

---

## 四、2026 年底主流 AI 编程代理横向全景

| 工具 | 核心形态 | 开源协议 | 模型生态 | 架构亮点与定位 |
|------|---------|---------|---------|---------------|
| **Claude Code** (`v2.1.326`) | 终端 CLI / IDE / Desktop / Web | 闭源商业 | Anthropic Claude 5.5 家族 & Fable 5.2（支持 Bedrock / Vertex / Foundry） | 复杂多文件重构与长程自治标杆；支持 Code Mode 批量脚本工具调用、Agent Teams、Cloud Scheduled Tasks 与 Channels |
| **OpenAI Codex** | 终端 CLI / Desktop / IDE / Cloud | CLI 开源（Apache 2.0） | OpenAI GPT-5.5 / 6.1 / Codex 系列 | 跨 macOS / Linux / Windows 原生 OS 沙箱、分层 `AGENTS.md` 指令链、与 ChatGPT 订阅及云端异步任务深度打通 |
| **Cursor** | 独立 AI IDE（VS Code Fork）/ Cloud / CLI | 闭源商业 | 多模型（Claude / OpenAI / Gemini + 自研 Composer 2.5） | 业界标杆级多文件视觉化 Diff 交互；支持隔离 Git Worktree 云端后台代理、手机端 Remote Control 与 Origin 应用市场 |
| **GitHub Copilot** | 多 IDE 插件（VS Code / JetBrains 等）+ GitHub.com | 闭源商业 | 多模型 + **原生托管运行 Claude 与 Codex 代理** | 与 GitHub Issue-to-PR 云端工作流无缝融合（原实验项目 Copilot Workspace 已演进为原生 Copilot Coding Agent），统一企业 AI Credits 账单治理 |
| **Gemini CLI / Antigravity** | 终端 CLI / Agentic IDE | Gemini CLI 开源（Apache 2.0） | Google Gemini 系列（原生 1M+ 上下文） | 2026 年 6 月起双线分流：开源 Gemini CLI 聚焦企业级 Code Assist / Vertex AI；个人开发者工作流迁移至多模态代理平台 Antigravity |
| **OpenCode** | 终端 TUI / Desktop / IDE | 开源（MIT） | 75+ 云端厂商（Models.dev）+ Ollama / vLLM 本地模型 | 社区高人气零厂商锁定开源编程代理，内置 LSP 诊断、子代理、MCP 与 Agent Skills 支持 |
| **OpenHands** | Web Agent Canvas / CLI / SDK / 自托管云 | 开源（MIT） | 100+ 模型（LiteLLM）+ **ACP 外部代理宿主** | 企业级多代理编排控制中心，可通过 ACP 在独立 Git Worktree 中并行调度 Claude Code、Codex、Gemini CLI 与自有代理 |

---

## 五、全网最新权威资源导航

- **Claude Code 官方文档与 Changelog**：https://code.claude.com/docs ｜ https://github.com/anthropics/claude-code/releases
- **Claude Platform API 与迁移指南（`2026-10-21`）**：https://docs.anthropic.com
- **Claude Agent SDK**：[TypeScript (`v0.3.326`)](https://github.com/anthropics/claude-agent-sdk-typescript) ｜ [Python (`v0.2.183`)](https://github.com/anthropics/claude-agent-sdk-python)
- **MCP 官方规范（`2026-07-28`）与官方注册中心**：https://modelcontextprotocol.io ｜ https://github.com/modelcontextprotocol/registry
- **Agentic AI Foundation (AAIF)**：https://aaif.io
- **开放代理标准**：[AGENTS.md](https://agents.md) ｜ [Agent Skills (`SKILL.md`)](https://agentskills.io) ｜ [Zed ACP 协议](https://zed.dev/acp)
