---
n: 19
chapter: "模块六 多代理与高级特性"
title: "Bridge & Remote — 跨设备协作"
generated_at: "2026-05-01T10:02:09.779151"
generated_by: "course-restructure-v2"
author: "anonymous"
---

# 第十九章：Bridge & Remote — 跨设备协作

![章节封面](images/ch19/hero.png)

> "Claude Code 不是终端工具——它是一个能跑在终端、Web、Mobile、SSH 上的同构 Agent。"
>
> 终端只是它最早的 frontend，不是它唯一的 frontend。

---

## 19.1 为什么需要 Bridge：claude.ai/code Web 应用的后端

把 Claude Code 当成"终端工具"看，是绝大多数用户的第一直觉，也是最大的一个误解。

打开 `https://claude.ai/code`，你会看到一个 Web IDE 风格的界面：左边是会话列表，中间是聊天框，右边是工具调用记录。它能读你的文件、跑你的命令、改你的代码——和你在终端里 `claude` 启动的体验一模一样。但你会想：浏览器跑不了 `BashTool`、写不了你本机的文件、读不了你 git 仓库——那 Web 应用是怎么做到这些的？

答案非常朴素：**Web 应用本身不做这些事——它把消息转发给一个跑在你本机的 Claude Code 进程，由它来执行**。这个"跑在本机、连着 Web 服务器、把 Web 客户端的消息搬运过来执行、再把结果搬运回去"的角色，就叫 **Bridge**(桥)。

### 终端模式 vs Web 模式：同构的核心

写一个 Web 应用最朴素的做法，是把整个 Agent 循环（包括工具调用）都搬到服务器上跑：浏览器只是一个聊天 UI，所有"读文件、跑命令"的脏活都由服务器代劳。但 Claude Code **没**走这条路，原因有四：

1. **代码必须在用户本机**：用户的 git 仓库、SSH 私钥、`.env` 文件、本地 Postgres，都不在 Anthropic 的服务器上。把它们传上服务器既不安全也做不到。
2. **工具的语义必须本地化**：`BashTool` 跑的是你本机的 shell；`FileEditTool` 写的是你本机的磁盘；`MCPTool` 连的是你本机配置的 MCP server。这些语义只能在本机执行。
3. **延迟必须低**：Agent 一次循环可能涉及十几次工具调用。如果每次工具调用都要 RTT 一次到 Anthropic 服务器，体验会卡到无法接受。
4. **离线可用性**：Anthropic 的服务可能短暂不可达，但用户的本机仍然能跑代码——Agent 循环必须能在本机完成，只把"模型推理"这一步外包给云端。

所以 Claude Code 的真正架构是：**Agent 循环跑在本机；模型推理跑在云端；UI 既可以是终端也可以是浏览器**。

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  浏览器 UI   │ ◄─────► │  Anthropic   │ ◄─────► │  本机 Bridge │
│  (Web 模式)  │  HTTPS  │   Server     │  HTTPS  │  (你的电脑)  │
└──────────────┘         └──────────────┘         └──────────────┘
                                                          │
                                                          ▼
                                                    ┌──────────┐
                                                    │ Agent 循环│
                                                    │ 工具执行  │
                                                    │ 模型 API  │
                                                    └──────────┘
```

终端模式下，**没有 Bridge**——你直接在本机看到一个 React 渲染的终端 UI，下面是 Agent 循环。Web 模式下，**Bridge 把消息从云端搬运到本机的 Agent 循环**，UI 渲染交给浏览器。两种模式共享**同一份 Agent 循环、同一份工具实现、同一份 Hook 系统**——这就是"同构"。

![Bridge 30 文件分组图（API 层 / 消息层 / 轮询层 / REPL 层 / 认证层 / UI 层），每组用不同颜色](images/ch19/02-img02.png)

### Bridge 不是远程执行

容易混淆的一点：Bridge **不是**"远程执行 Claude Code"。Claude Code 仍然在你本机跑，只是把 UI 换成了浏览器。换句话说：

- **错的理解**：Bridge 让浏览器远程操作你的电脑（像 VNC）。
- **对的理解**：Bridge 让你的浏览器和你电脑里那个 Claude Code 进程"对话"。Agent 循环还是在你本机跑。

这两者的差别是：远程执行模式下，关掉本机进程就什么都没了；Bridge 模式下，**关掉浏览器，本机进程仍然在跑**——你下次打开 Web，还能继续这次对话。

### Bridge 系统的定位

如果把整个 Claude Code 看作"Agent 内核 + 多种 frontend"的双层架构：

- **内核**：`src/core/` 的 query loop、`src/tools/` 的 40 个工具、`src/services/claude.ts` 的模型客户端、`src/permissions/` 的 5 种权限模式。
- **Frontend 1**：终端 UI（`src/repl/`、`src/screens/`、`src/components/` 那一套 React + Ink 渲染）。
- **Frontend 2**：Web UI（claude.ai/code，运行在浏览器里）。
- **Frontend 3**：Mobile App（iOS/Android Claude App）。
- **Frontend 4**：SSH（你 SSH 到远程服务器再启动一个 Claude Code）。

**Bridge 就是 Frontend 2 和 Frontend 3 在本机这一侧的"代驾"**——它代替终端 UI 把消息送进 Agent 内核，又把 Agent 内核的输出代替终端 UI 推送回浏览器/手机。

理解这层定位之后，本章接下来要讲的所有东西——12,613 行 Bridge、1,127 行 Remote、1,229 行 Voice、SSH、Teleport、UpstreamProxy——都是围绕"Agent 内核怎么跨设备协作"这一个问题展开的。

---

## 19.2 Bridge 系统总览（src/bridge/，30 个文件，12,613 行）

`src/bridge/` 目录是 Claude Code 体量最大的子系统之一——**30 个 TypeScript 文件，12,613 行代码**（数据见 `docs/canonical-numbers.md` 第 18 项）。这个体量是因为 Bridge 不是"一个把消息从 A 搬到 B 的简单转发器"，它要解决的问题包括：

- 怎么用 OAuth 认证一个 Web 客户端，并把"这个 Web 客户端"和"本机这个 Agent"绑定起来；
- 怎么在不可靠的 HTTP 长轮询上实现"近实时"的双向消息流；
- 怎么处理 Web 客户端和 Agent 循环之间的速率差（Agent 一秒能产生几十条消息，Web 客户端一秒只 poll 一次）；
- 怎么处理"Agent 正在跑，但用户合上了笔记本"这种场景；
- 怎么把 Web 客户端的工具确认、撤销、附件上传，原汁原味地翻译成本机 Agent 循环能理解的事件。

### 所有 30 个文件的职责

按职责分组列一下（行数来自 `wc -l`）：

**API 层（与 Anthropic Server 通讯）**

| 文件 | 行数 | 职责 |
|---|---|---|
| `bridgeApi.ts` | 539 | Bridge HTTP client，封装 axios 请求、401 重试、token 校验 |
| `codeSessionApi.ts` | 168 | Code Session API（创建/恢复/列表）|
| `bridgeConfig.ts` | 48 | Bridge 配置加载入口 |
| `envLessBridgeConfig.ts` | 165 | 不依赖环境变量的 Bridge 配置（用于 daemon 启动）|
| `bridgeEnabled.ts` | 202 | "当前会话是否处于 Bridge 模式"判断 |

**主循环与会话生命周期**

| 文件 | 行数 | 职责 |
|---|---|---|
| `bridgeMain.ts` | 2,999 | Bridge 主进程入口（最大文件），承载主轮询循环 |
| `sessionRunner.ts` | 550 | 单个会话的运行器，绑定 Agent 循环 |
| `createSession.ts` | 384 | 会话创建流程（注册到 Server、绑定本机进程）|
| `bridgePointer.ts` | 210 | "Pointer"——指示 Server 当前消息读到哪条的游标 |
| `sessionIdCompat.ts` | 57 | session ID 历史命名兼容（早期用过其它命名）|

**消息层**

| 文件 | 行数 | 职责 |
|---|---|---|
| `bridgeMessaging.ts` | 461 | 出站消息：Agent → Server 的发送队列与批量上传 |
| `inboundMessages.ts` | 80 | 入站消息：Server → Agent 的解析（user message）|
| `inboundAttachments.ts` | 175 | 入站附件：Web 客户端上传的图片/文件 |
| `flushGate.ts` | 71 | 状态机：在批量 flush 期间阻塞新消息入队 |

**轮询/调度**

| 文件 | 行数 | 职责 |
|---|---|---|
| `pollConfig.ts` | 110 | 轮询间隔配置加载与校验 |
| `pollConfigDefaults.ts` | 82 | 默认间隔（at-capacity / seek-work / heartbeat）|
| `capacityWake.ts` | 56 | 容量唤醒：在 at-capacity 时被外部信号叫醒 |

**REPL Bridge（高级模式）**

| 文件 | 行数 | 职责 |
|---|---|---|
| `replBridge.ts` | 2,406 | REPL Bridge 主入口（第二大文件），把本机 REPL 暴露给云端 |
| `initReplBridge.ts` | 569 | REPL Bridge 启动初始化 |
| `replBridgeTransport.ts` | 370 | REPL Bridge 传输层（WebSocket/HTTP 双通道）|
| `replBridgeHandle.ts` | 36 | REPL Bridge 句柄，外部调用入口 |
| `remoteBridgeCore.ts` | 1,008 | Remote Bridge 公共核心（被多个 Bridge 模式共用）|

**认证与安全**

| 文件 | 行数 | 职责 |
|---|---|---|
| `jwtUtils.ts` | 256 | JWT 解码、过期判断、duration 格式化 |
| `trustedDevice.ts` | 210 | 受信任设备 token，对接 ELEVATED 安全等级 |
| `workSecret.ts` | 127 | "Work secret"——临时一次性密钥，绑定单次任务 |

**调试与 UI 反馈**

| 文件 | 行数 | 职责 |
|---|---|---|
| `bridgeDebug.ts` | 135 | Bridge 调试输出格式化 |
| `bridgeStatusUtil.ts` | 163 | Bridge 状态字串（用于状态栏渲染）|
| `bridgeUI.ts` | 530 | Bridge 模式下的终端反馈渲染（在终端里也能看到 Web 用户在干嘛）|
| `bridgePermissionCallbacks.ts` | 43 | 权限回调注册接入 |
| `debugUtils.ts` | 141 | 公共调试工具 |
| `types.ts` | 262 | 公共类型定义 |

合计 30 个文件、**12,613 行**——和 `docs/canonical-numbers.md` 第 18 项一致。

![capacityWake 三路信号合并示意图：outerSignal + wakeController → mergedSignal → 谁先 abort 谁触发](images/ch19/04-img04.png)

### 文件分组的内在结构

如果把上面 30 个文件按"调用关系"重新组织，会发现一个非常清晰的层级：

```
最上层：bridgeMain.ts（主进程）
        ├── 调用 → createSession.ts（建立会话）
        ├── 调用 → sessionRunner.ts（跑一个会话）
        │           ├── 调用 → bridgeMessaging.ts（发出站消息）
        │           ├── 调用 → inboundMessages.ts（处理入站消息）
        │           └── 调用 → flushGate.ts（节流）
        ├── 使用 → capacityWake.ts（在 at-capacity 时阻塞/唤醒）
        ├── 使用 → pollConfig.ts（轮询间隔）
        └── 通过 bridgeApi.ts 与 Server 通讯
                  ├── 使用 → jwtUtils.ts（JWT 解析）
                  └── 使用 → trustedDevice.ts（设备信任头）
```

`replBridge.ts` 是另一条独立的"REPL 镜像"链路——把本机 REPL 直接暴露给云端，让云端的 Web 客户端能看到 REPL 的实时输出。它和 `bridgeMain.ts` 不冲突，可以并行存在。

> **本章策略**：12,613 行不可能逐文件读完。我们挑出**最具代表性的 8 个文件**展开（`jwtUtils.ts` / `trustedDevice.ts` / `inboundMessages.ts` / `inboundAttachments.ts` / `flushGate.ts` / `pollConfig.ts` / `pollConfigDefaults.ts` / `capacityWake.ts`），用它们勾画出 Bridge 的"消息层 + 轮询层 + 认证层"骨架；`bridgeMain.ts` 和 `replBridge.ts` 这两个大文件留作"自己读源码时的参考点"，不试图在一章里穷尽。

### 与 Agent 内核的接口

Bridge 不是从零跑一个 Agent——它**复用** `src/core/` 那个 query loop，只是替换了 frontend：

- 在终端模式下，frontend 是 `src/repl/` + `src/screens/` 的 React + Ink。
- 在 Bridge 模式下，frontend 是 `bridgeMain.ts` 起的轮询循环 + `bridgeMessaging.ts` 的发送/接收队列。

两边都把 user message 送进同一个 query loop，都从同一个 query loop 里收 assistant message，区别只在"消息从哪来、往哪送"。这种设计的好处是 Agent 内核不知道"自己被谁调用"——它面向一个抽象的 message stream 工作。这也是为什么后面我们会看到 SSH、Teleport、Direct Connect 这些"看似不一样"的模式都能复用同一份内核。

---

## 19.3 JWT 认证与 trustedDevice

Web 客户端要怎么证明"我是 Anthropic 颁发给你的合法客户端，不是攻击者伪造的"？答案是 **JWT（JSON Web Token）**——一段由 Anthropic Server 签名的、带过期时间的、带 claims 的字符串。

### Bridge JWT 的生命周期

`src/bridge/jwtUtils.ts:21` 的 `decodeJwtPayload` 函数告诉了我们 Bridge JWT 的形态：

```typescript
// src/bridge/jwtUtils.ts:21-32
export function decodeJwtPayload(token: string): unknown | null {
  const jwt = token.startsWith('sk-ant-si-')
    ? token.slice('sk-ant-si-'.length)
    : token
  const parts = jwt.split('.')
  if (parts.length !== 3 || !parts[1]) return null
  try {
    return jsonParse(Buffer.from(parts[1], 'base64url').toString('utf8'))
  } catch {
    return null
  }
}
```

注意几个细节：

1. **前缀剥离**：Bridge token 带前缀 `sk-ant-si-`（"si" 应该是 session-ingress 的缩写）。这种前缀是 Anthropic 内部统一的命名习惯（用户级 API key 是 `sk-ant-api-`，Bridge session 是 `sk-ant-si-`），方便日志里一眼区分。
2. **不验证签名**：`decodeJwtPayload` **不**验证签名——它只解 base64 看 payload。原因是 Bridge 的 JWT 来自 Server，CLI 拿到时是已经被 Server 校验过一次的；CLI 只需要从 payload 里读 claims（如 `exp`、`session_id`），不必自己再验证一次。**真正的信任锚是 Server**。
3. **格式严格**：必须三段、中间段必须存在，否则返回 `null`——任何畸形 JWT 都会被拒。

`jwtUtils.ts` 还提供 `formatDuration`（把毫秒格式化成 "5m 30s"）、过期判断、剩余时长计算等工具函数——它们都是给日志/调试用的，让你在终端能看到 "JWT 还有 12m 30s 过期" 这种提示。

### JWT 在 Web Session 中的作用

Web Session 的"JWT 流"是这样的（简化版）：

```
1. 用户在 claude.ai/code 上点"启动一个 Claude Code session"
2. Web 前端 → Anthropic Server：POST /api/code/sessions
3. Server 决定"哪台用户的本机 Bridge 进程接这个 session"，
   颁发一个 JWT：sk-ant-si-xxx.yyy.zzz
4. Server 通过双向通道（具体见 capacityWake/poll 那一节）把 JWT 推到本机 Bridge
5. 本机 Bridge 把 JWT 存进内存，每次调用 Server API 时带上：
   Authorization: Bearer sk-ant-si-xxx.yyy.zzz
6. Server 校验签名 + 过期时间 + session_id 的归属关系
7. 通过则把当前会话的消息推/拉给本机 Bridge
```

`bridgeApi.ts` 里 `getAccessToken` 这个 dep 就是用来在每次请求时实时拿最新 JWT 的（见 `src/bridge/bridgeApi.ts:13`）——因为 JWT 会过期、会被 Server 主动刷新。

### 401 重试与 token 刷新

JWT 不是永久的。当本机 Bridge 拿一个**已经过期**的 JWT 去调 Server，Server 会返回 401。`bridgeApi.ts` 的 deps 接口里有这样一段（`src/bridge/bridgeApi.ts:18-26`）：

```typescript
/**
 * Called on 401 to attempt OAuth token refresh. Returns true if refreshed,
 * in which case the request is retried once. Injected because
 * handleOAuth401Error from utils/auth.ts transitively pulls in config.ts →
 * file.ts → permissions/filesystem.ts → sessionStorage.ts → commands.ts
 * (~1300 modules). Daemon callers using env-var tokens omit this — their
 * tokens don't refresh, so 401 goes straight to BridgeFatalError.
 */
onAuth401?: (staleAccessToken: string) => Promise<boolean>
```

这段注释信息量很大：

- **注入而不直接调用**：`handleOAuth401Error` 牵涉 1300 个模块（permissions、sessionStorage、commands），如果直接 import 进 Bridge，就会把 Bridge 的 cold start 时间拖死。所以走依赖注入——主进程在初始化 Bridge 时把这个函数传进去，daemon 模式下不传（daemon 用环境变量里的 token，不会刷新）。
- **失败 fail-fast**：daemon 没 onAuth401，401 直接抛 `BridgeFatalError`——daemon 模式下宁可整个停掉也不要"半死不活"。

### trustedDevice：ELEVATED 安全层

`trustedDevice.ts` 解决另一个问题：怎么让 Server 知道"这个 Bridge token 是从一台 **被该用户授权过的设备** 上发出的"？

Bridge session 在 Server 侧的 SecurityTier 是 **ELEVATED**——比普通 API key 更高一档。`trustedDevice.ts` 的 doc comment（`src/bridge/trustedDevice.ts:15-30`）解释了这个机制：

> Bridge sessions have SecurityTier=ELEVATED on the server (CCR v2). The server gates ConnectBridgeWorker on its own flag (sessions_elevated_auth_enforcement in Anthropic Main); this CLI-side flag controls whether the CLI sends X-Trusted-Device-Token at all. Two flags so rollout can be staged: flip CLI-side first (headers start flowing, server still no-ops), then flip server-side.

要点：

1. **两个 flag 双开**：CLI 侧（这里实现）控制"是否发 `X-Trusted-Device-Token` 头"，Server 侧控制"是否强制要求该头"。这种"两边各自开关"是 Anthropic 大型基础设施变更的标准 rollout 模式——CLI 先全量铺开发头，Server 仍然 no-op，等头流量稳了再开 Server enforcement。
2. **ELEVATED 的语义**：Bridge session 能访问用户本机的代码、能跑 shell——比"普通问答"危险得多，所以需要更高的认证保证。
3. **设备信任链**：用户首次启动 Bridge 时，本机会向 Server `POST /auth/trusted_devices` 注册自己（设备指纹 + OAuth token）。注册成功后 Server 颁发 `trustedDeviceToken`，存进 secure storage。后续每次 Bridge API 调用都把这个 token 通过 `X-Trusted-Device-Token` 头送出——Server 校验该 token 有效，才放行 ELEVATED 操作。

`bridgeApi.ts` 是这样消费 trustedDevice 的（`src/bridge/bridgeApi.ts:28-35`）：

```typescript
/**
 * Returns the trusted device token to send as X-Trusted-Device-Token on
 * bridge API calls. Bridge sessions have SecurityTier=ELEVATED on the
 * server (CCR v2); when the server's enforcement flag is on,
 * ConnectBridgeWorker requires a trusted device at JWT-issuance.
 */
getTrustedDeviceToken?: () => string | undefined
```

注意这里也是依赖注入而不是 import——和 `onAuth401` 一样，避免把 secureStorage 那一坨依赖拖进 Bridge 模块。

### 设备信任链建立流程

完整流程（用户视角）：

```
首次启动 Bridge:
  1. Claude Code CLI 检查本地 secureStorage 是否有 trustedDeviceToken
  2. 没有 → 弹出登录提示（OAuth 浏览器流程）
  3. OAuth 通过 → 拿到 access_token
  4. POST /auth/trusted_devices （body: device fingerprint + access_token）
  5. Server 返回 trustedDeviceToken → 存进 secureStorage
  6. 所有 Bridge API 调用现在都带 X-Trusted-Device-Token 头

后续启动:
  1. 检查 secureStorage 有 trustedDeviceToken → 直接用
  2. （可选）background refresh 该 token 的有效期
```

设备信任链一旦建立，**不绑死任何一台 Web 客户端**——你可以从任意浏览器打开 claude.ai/code，都能连到这台本机 Bridge，因为信任的对象是"设备"，不是"浏览器会话"。

> **教学要点**：JWT + trustedDevice 是 Bridge 安全的两根支柱。JWT 解决"这条消息是来自合法 session 的"，trustedDevice 解决"这台 CLI 跑在用户授权过的机器上"。两者缺一不可——单 JWT 容易被偷（攻击者拿到 token 就能伪造），单 trustedDevice 容易被滥用（一台机器签所有人）。

---

## 19.4 inboundMessages 与 inboundAttachments

Web 客户端发出来的消息怎么走进 Agent 循环？答案在 `src/bridge/inboundMessages.ts` 和 `src/bridge/inboundAttachments.ts` 这两个加起来 255 行的小文件里。

### inboundMessages：从 SDKMessage 到 Agent 输入

`inboundMessages.ts` 只有 80 行，但承担了一个关键角色——**Web 客户端 → Agent 的格式翻译器**。

Web 客户端发来的消息是一个 `SDKMessage`（Claude Agent SDK 定义的统一格式），里面可能包含：

- 纯字符串：`"帮我看看这个 bug"`
- ContentBlockParam 数组：包含文本块 + 图片块（用户拖了张截图进 Web 输入框）
- 元数据：`uuid`、`timestamp`、`session_id` 等

Agent 循环需要的是**纯净的 `content` + 一个可选的 UUID**——元数据它自己会从 session 上下文里取。所以 `extractInboundMessageFields` 做的事就是：

```typescript
// src/bridge/inboundMessages.ts:21-25
export function extractInboundMessageFields(
  msg: SDKMessage,
):
  | { content: string | Array<ContentBlockParam>; uuid: UUID | undefined }
  | undefined {
```

它只做三件事：

1. **过滤非 user 消息**：Bridge 的入站只关心 user message（system / assistant / result 这些不应该从 Web 客户端反向流入）。返回 `undefined` = 跳过。
2. **解出 content**：可能是 string，也可能是 ContentBlockParam[]——两种都直接透传给 Agent 循环（Agent 循环本身就两种都接）。
3. **解出 UUID**：如果 Web 客户端给了 message UUID，透传；没给，让 Agent 循环自己生成。

### 一个真实的兼容性陷阱

`inboundMessages.ts` 的 doc comment 里有这样一段（`src/bridge/inboundMessages.ts:12-15`）：

> Normalizes image blocks from bridge clients that may use camelCase `mediaType` instead of snake_case `media_type` (mobile-apps#5825).

这里能看到一个真实的 frontend / backend 不一致案例：

- **Anthropic SDK 的标准**：图片块的字段叫 `media_type`（snake_case，对齐 Anthropic API）
- **Mobile App 早期实现**：写成了 `mediaType`（camelCase，对齐 Swift/Kotlin 习惯）

这种不一致在 Mobile App 上线后才暴露——issue 编号是 `mobile-apps#5825`。Bridge 端不能要求 Mobile 重新发版（用户量大、发版慢），只能在入口处兼容两种命名。这也是为什么 `inboundMessages.ts` 哪怕只有 80 行，也要做归一化——它是 Web/Mobile 两条链路汇入 Agent 内核的咽喉。

> **教学点**：跨设备协议的"鲁棒性"不是来自严格的协议规范，而是来自**入口处的归一化层**。任何一个跨进程/跨语言的边界，都该有一个像 `inboundMessages.ts` 这样的"翻译器 + 净化器"——它能挡住下游所有"上游版本不一致"导致的 bug。

### inboundAttachments：图片附件的处理

`inboundAttachments.ts` 175 行，专门处理图片附件——为什么图片要单独一个文件？

因为图片附件涉及几个其他文本消息没有的问题：

1. **Base64 vs URL**：Web 客户端可能直接发 Base64 编码的图片数据（小图），也可能发"图片已经上传到 Anthropic Files API，这是 file_id"（大图）。Bridge 要适配两种。
2. **格式探测**：Web 客户端发的 Base64 可能没带 `media_type`，需要从 base64 头几个字节探测 MIME（PNG 的 `\x89PNG` magic、JPEG 的 `\xff\xd8`）。`utils/imageResizer.ts` 提供了 `detectImageFormatFromBase64`，被 `inboundAttachments.ts` 直接使用（见 `src/bridge/inboundMessages.ts:8`）。
3. **大小限制**：图片超过一定大小会被 Anthropic 模型拒绝。Bridge 要么本地 resize、要么向上层抛错。
4. **格式归一化**：和文本一样，camelCase / snake_case 兼容；除此之外还要把 `image_url`、`base64` 两种来源统一成 Anthropic API 接受的格式。

### Web 客户端 → Server 的消息上行流程

完整路径：

```
1. 用户在 claude.ai/code 输入框打字 + 拖一张截图，按 Enter
2. Web 前端构造 SDKMessage:
     {
       type: 'user',
       content: [
         { type: 'text', text: '帮我看看这个 bug' },
         { type: 'image', source: { type: 'base64', mediaType: 'image/png', data: '...' } }
                                            ^^^ camelCase（Mobile/早期 Web）
       ],
       uuid: 'abc-...'
     }
3. Web 前端 → Anthropic Server: POST /api/code/sessions/:id/messages
4. Server 校验 JWT + 把消息存进当前 session 的 inbox queue
5. 本机 Bridge 通过 poll 循环拉取 → bridgeApi.fetchInbound(...)
6. inboundMessages.ts:extractInboundMessageFields(msg) 抽出 { content, uuid }
7. inboundAttachments.ts: 处理 image block，归一化 mediaType → media_type
8. Agent 循环:enqueueUserMessage({ content, uuid })
9. Agent 循环开始一次新的 query → 调 model → 决定是否调工具 → 流式产出 assistant message
10. assistant message 经 bridgeMessaging.ts 反向送回 Server → Web 客户端
```

注意第 5 步：是**本机主动拉**，不是 Server 主动推——Bridge 用的是长轮询模型，理由我们在下一节讲。

---

## 19.5 flushGate 与 polling 配置

Bridge 的"消息搬运"不是一句话能讲清楚的——它要解决两个互相纠缠的问题：

1. **批量与实时的矛盾**：Agent 一次循环可能产生几十条消息（thinking / tool_use / tool_result / assistant 文本块）。把每条都单独发是高开销；批量发又会让 Web 客户端看到"卡一下、然后一批刷出"的卡顿体验。
2. **轮询频率与服务器负载的矛盾**：本机 Bridge 想第一时间知道 Web 客户端发了新消息，最好每 100ms poll 一次；但 Server 想保护自己，希望本机别那么频繁打它。

`flushGate.ts`（71 行）解决问题 1，`pollConfig.ts` + `pollConfigDefaults.ts`（合计 192 行）解决问题 2。

### flushGate：批量 flush 期间的写阻塞

`flushGate.ts` 是一个泛型状态机，doc comment 把它讲得非常清楚（`src/bridge/flushGate.ts:1-15`）：

> When a bridge session starts, historical messages are flushed to the server via a single HTTP POST. During that flush, new messages must be queued to prevent them from arriving at the server interleaved with the historical messages.
>
> Lifecycle:
>   start() → enqueue() returns true, items are queued
>   end()   → returns queued items for draining, enqueue() returns false
>   drop()  → discards queued items (permanent transport close)
>   deactivate() → clears active flag without dropping items

翻译人话：

```
场景：Bridge 启动时要把历史消息（之前会话的 assistant 输出）一次性 POST 给 Server，
让 Web 客户端能恢复"接着之前的对话往下"的状态。

问题：在这个 POST 还没完成时，Agent 循环又在继续产出新消息（用户没等历史 flush 完
就发新消息了，或后台 task 还在跑）。新消息如果直接 POST 出去，可能会比历史 flush
先到 Server，导致 Server 那边消息顺序错乱。

解决：FlushGate 当门卫——
  flush 开始 → start() → 后续所有 enqueue 进入排队
  flush 结束 → end() → 把排队的全部一次性吐出来，让调用方决定怎么发
```

### FlushGate 的 4 个生命周期方法

```typescript
// src/bridge/flushGate.ts:16-19
export class FlushGate<T> {
  private _active = false
  private _pending: T[] = []
```

- **`start()`**：开门收件，`_active = true`。所有 enqueue 进 `_pending`。
- **`enqueue(item)`**：如果 active 返回 true（已收下），否则返回 false（调用方自己处理）。
- **`end()`**：返回积压的 `_pending` 数组，置 `_active = false`，调用方拿到这批"在 flush 期间被挡住的消息"决定怎么处理（通常是合并发出去）。
- **`drop()`**：传输永久关闭时用，丢弃所有 pending（不会再有 transport 来发了）。
- **`deactivate()`**：传输被替换时用，清 active 但保留 pending（让新 transport 来 drain）。

### deactivate 和 drop 的微妙区别

这是一个"读源码读出味道"的细节。两个方法看起来都是"关闭",但语义完全不同：

- **drop**：永久关闭。比如 session 结束、Bridge 退出——pending 里的消息再也没人接了，必须丢。
- **deactivate**：临时关闭。比如长轮询 transport 因为网络抖动重连——消息不能丢（用户的输入要保留），但当前 transport 不再适合 drain（它马上要被替换）。新 transport 起来后会调 `start()` 重新 active，pending 还在。

这种"丢/不丢"的二分在大型分布式系统里是个常见话题——通常一不小心就会丢东西，或者反过来，一不小心就会重复发送（idempotency 失败）。FlushGate 用两个明确命名的方法把这个区别外显出来，逼调用方在两种语义之间显式选择。

### pollConfig：长轮询频率的三档配置

Bridge 用 HTTP 长轮询（不是 WebSocket）拉取 Server 的新消息——这有几个原因：HTTP 穿透代理/防火墙更容易、运维更简单、断线重连成本低。但长轮询的关键是"间隔多久 poll 一次"。`pollConfigDefaults.ts` 定义了**三档**：

```
 atCapacity   — Agent 正在跑（已经在处理一个 query），偶尔确认下"用户还想我别取消吗"
 seekWork     — Agent 空闲，主动找新任务
 heartbeat    — 心跳，"我还活着"探测
```

`pollConfig.ts:15-26` 是配置 schema 的一段精彩注释：

> .min(100) on the seek-work intervals restores the old Math.max(..., 100) defense-in-depth floor against fat-fingered GrowthBook values. Unlike a clamp, Zod rejects the whole object on violation — a config with one bad field falls back to DEFAULT_POLL_CONFIG entirely rather than being partially trusted.

这段讲了三件事：

1. **GrowthBook 控制**：这些间隔不是硬编码——它们从 GrowthBook（Anthropic 的 feature flag 系统）拿。运维可以不发版就调整轮询频率。
2. **fat-finger 保护**：但 GrowthBook 是人配的，万一手抖配了 `10`（毫秒）会怎么样？Bridge 会以 100Hz 频率打 Server，DB 会冒烟。所以 schema 强制 `.min(100)` ——任何 < 100ms 的值都会让整个 schema 校验失败。
3. **一坏全坏 vs 局部 clamp**：传统做法是 `Math.max(value, 100)`——你输入 10，我帮你 clamp 到 100，继续跑。Zod 这里反过来：**任何一个字段坏，整个配置作废，回退到 DEFAULT_POLL_CONFIG**。这种"一坏全坏"看起来更激进，但更安全——你不知道用户配错的字段是不是和其他字段有耦合关系（比如 atCapacity 配错的同时 heartbeat 也配错），局部 clamp 反而可能让你跑在一个"半坏"配置上。

### at-capacity 的 0-or-≥100 设计

```typescript
// src/bridge/pollConfig.ts:21-25
// The at_capacity intervals use a 0-or-≥100 refinement: 0 means "disabled"
// (heartbeat-only mode), ≥100 is the fat-finger floor. Values 1–99 are
// rejected so unit confusion (ops thinks seconds, enters 10) doesn't poll
// every 10ms against the VerifyEnvironmentSecretAuth DB path.
```

at-capacity 间隔有特殊语义：**0 = 禁用，≥100 = 启用**。1-99 被显式拒绝——因为这个区间唯一可能产生的方式就是 ops 把秒数误填成毫秒（"10 秒" → 写了 `10`），这是个该报错的配置错误，不是该 silently clamp 的边角值。

这种"在 type system 层面强制单位语义"是大型系统少有的好习惯。大部分系统对单位是默认信任的——"我相信你写的是 ms"，结果某天某人输入了 "10"，整个集群往 DB 打 100Hz。

### object-level refine：互锁约束

`pollConfig.ts:23-26` 还有一个细节：

> The object-level refines require at least one at-capacity liveness mechanism enabled: heartbeat OR the relevant poll interval. Without this, the hb=0, atCapMs=0 drift config (ops disables heartbeat without restoring at_capacity) falls through every throttle site with no sleep — tight-looping /poll at HTTP-round-trip speed.

意思是 schema 不止校验单字段，还跨字段做"互锁约束"：

```
约束：heartbeat OR at_capacity 至少一个 > 0
否则：config 整体作废
```

为什么要这种约束？因为这两个字段**不是独立的**——它们都是"在 Agent 满载时控制 poll 频率"的机制，必须至少有一个起作用。如果两个都是 0，poll 循环根本就没有 sleep，会以 HTTP RTT 的速度（10-50ms）疯狂打 Server。这个 bug 之前真的发生过——ops 在 GrowthBook 改 heartbeat 配置时把 at_capacity 也清了，导致一个机房的 CLI 集体往 Server 打到限流。这个 refine 就是事故复盘后加的。

> **教学点**：FlushGate 教我们"如何在 flush 边界保持消息顺序"，pollConfig 教我们"如何用 schema 校验把运维配置错误挡在外面"。这两个文件加起来 263 行，但每一行都是被生产事故浇出来的经验。

![Voice 录音 + STT + keyterms 的数据流图：麦克风 → PCM → WebSocket → Server STT → 转录文本 + keyterms → 填回 REPL](images/ch19/08-img08.png)

### pollConfig 三档间隔的语义图

补充一张图说明这三档怎么配合：

```
状态：seek-work（Agent 空闲）
        间隔：seekWorkIntervalMs（默认几百毫秒）
        目的：找到新任务就立刻开干

状态：at-capacity（Agent 满载）
        间隔：atCapacityIntervalMs（默认数秒）
        目的：偶尔检查"用户是否取消"，主要靠 capacityWake 提前唤醒

状态：所有 idle 时段
        间隔：heartbeatMs（30s 量级）
        目的：让 Server 知道这个 Bridge 还活着，Server 才不会标记 stale
```

三档之间的切换是**事件驱动**的：

- Agent 接到新工作 → 切到 at-capacity；
- Agent 完成最后一个工作 → 切到 seek-work；
- 一段时间没新活 → heartbeat 维持长连接的"假活"。

### 与 Server 端的速率限制配合

Server 一侧也有自己的限流。Bridge 和 Server 的轮询节奏需要对齐：

- Server 期望客户端每秒最多 10 次 poll；
- Bridge 默认配置 100ms 间隔（最快可能达到 10Hz）；
- 但 capacityWake 只在事件触发时才提前——不会持续打 10Hz。

这种"客户端尊重 Server 限流"的契约不是写在代码里的硬上限，而是通过 **GrowthBook 默认值 + 事件驱动的提前唤醒**自然达到的。任何一方违约（运维配错、容量唤醒疯狂触发）都会被监控告警捕获。

---

## 19.6 capacityWake（容量唤醒机制）

`capacityWake.ts` 一共 56 行——是 Bridge 目录里最小的几个文件之一。但它解决的问题非常微妙：**当 Agent "满载"时，poll 循环要 sleep 等容量空出来；但又不能死等——容量空出来的瞬间必须立刻醒来。**

### 满载等待的两难

想象一个场景：Web 客户端发了一个长任务给本机 Bridge，Agent 现在正在跑这个 query，每个 worker slot 都被占用（"at capacity"）。这时候本机不需要 poll Server 拉新消息——拉了也吃不下。但又不能完全断开——还得：

1. 监听用户的取消信号（用户在 Web 上点了"停止"）；
2. 监听容量释放信号（这次 query 完了，可以接下一个）；
3. 监听整体退出信号（用户关了 CLI、收到 SIGTERM）。

朴素的做法是开一个 `setInterval(checkCapacity, 1000)`。但这有两个问题：

- **延迟**：容量释放后最多还要等 1 秒才能恢复 poll；
- **资源**：永远在跑的 timer，CLI 想干净退出还得清它。

`capacityWake.ts` 的方案是：**用 AbortSignal 模拟"可重置的 wakeup"**。

### 三个 AbortSignal 的合流

`capacityWake.ts:11-26` 定义了接口：

```typescript
// src/bridge/capacityWake.ts:11-26
export type CapacitySignal = { signal: AbortSignal; cleanup: () => void }

export type CapacityWake = {
  signal(): CapacitySignal
  wake(): void
}

export function createCapacityWake(outerSignal: AbortSignal): CapacityWake {
  let wakeController = new AbortController()
```

注意 `wakeController` 的位置——它是一个**可变绑定（let）**，不是 const。每次 wake 之后会被换成一个新的 controller：

```typescript
// src/bridge/capacityWake.ts:30-33
function wake(): void {
  wakeController.abort()
  wakeController = new AbortController()
}
```

这是关键技巧：**用"换实例"模拟"重置"**。AbortController 的 abort 是单向的（一旦 abort 就永远是 aborted 状态），所以 wake 一次就要换一个新实例，下次 sleep 才能从一个 fresh controller 拿一个新的 signal。

### 三路信号合并

`capacityWake.ts:35-53` 实现了"任意一路 abort 就 abort 整体"的合并器：

```typescript
function signal(): CapacitySignal {
  const merged = new AbortController()
  const abort = (): void => merged.abort()
  if (outerSignal.aborted || wakeController.signal.aborted) {
    merged.abort()
    return { signal: merged.signal, cleanup: () => {} }
  }
  outerSignal.addEventListener('abort', abort, { once: true })
  const capSig = wakeController.signal
  capSig.addEventListener('abort', abort, { once: true })
  return {
    signal: merged.signal,
    cleanup: () => {
      outerSignal.removeEventListener('abort', abort)
      capSig.removeEventListener('abort', abort)
    },
  }
}
```

逻辑：

1. 创建一个 `merged` AbortController；
2. 任何一路（outerSignal = 整体退出、wakeController = 容量释放）abort，都会触发 merged abort；
3. 检查初始状态——如果有一路已经 aborted，直接 abort merged，cleanup 是 no-op；
4. 没 aborted 就挂监听，并返回一个 cleanup 用来在 sleep 正常结束时清监听（防止内存泄漏）。

### poll 循环用 capacityWake

伪代码（基于 doc comment）：

```typescript
// 在 bridgeMain.ts 或 replBridge.ts 的 poll loop 中:
while (!outerSignal.aborted) {
  if (atCapacity()) {
    const { signal, cleanup } = capacityWake.signal()
    try {
      await sleep(atCapacityIntervalMs, { signal })
    } catch {
      // signal aborted —— 三种可能：outer 退出、wake() 被调、超时
    } finally {
      cleanup()
    }
    continue
  }
  // 不 at capacity，正常 poll
  await pollServer()
}

// 别的地方调用 capacityWake.wake() 来唤醒：
sessionEndedEmitter.on('done', () => capacityWake.wake())
```

任意一路触发 abort 都会让 sleep 提前返回，循环立刻重新检查 capacity。等 sleep 自然结束，cleanup 回收监听器——不留内存。

![跨设备工作流时间轴：08:30 桌面→10:00 手机→14:00 SSH→18:00 Dispatch，箭头标注用了哪些子系统](images/ch19/14-img14.png)

### 为什么不直接用 setTimeout + clearTimeout

技术上 `setTimeout(check, ms)` + `clearTimeout` 也能做。但 capacityWake 用 AbortSignal 有几个明显优点：

1. **统一的取消语义**：Node.js / Web Standard 里所有现代 async API（`fetch`、`stream`、`fs.readFile`）都接受 AbortSignal。把"我要被取消"的语义统一在 signal 上，调用方写代码更一致。
2. **可组合**：`AbortSignal.any([sig1, sig2])` 是标准方法，可以无限组合。setTimeout 没有这种结构性能力。
3. **生命周期挂载**：cleanup 函数被显式返回——调用方知道"什么时候我应该清"。setTimeout 的 handle 容易忘记 clear，造成泄漏。

### 这 56 行的本质

`capacityWake.ts` 是 Claude Code 里"最小可行抽象"的典范——它解决了一个非常具体的问题（poll 循环 at-capacity 时的 sleep + 提前唤醒），用了语言原生的能力（AbortController），代码不到 60 行。

但它的价值在于**消除重复**——doc comment 写道：

> Both replBridge.ts and bridgeMain.ts need to sleep while "at capacity" but wake early when either (a) the outer loop signal aborts (shutdown), or (b) capacity frees up (session done / transport lost). This module encapsulates the mutable wake-controller + two-signal merger that both poll loops previously duplicated byte-for-byte.

之前 `replBridge.ts`（2,406 行）和 `bridgeMain.ts`（2,999 行）各自维护一份完全相同的 wake-controller 逻辑，行为一致但代码复制粘贴。`capacityWake.ts` 提取出来之后，两边都改成 `import { createCapacityWake }` ——以后哪怕逻辑要改，只改一份。

> **教学点**：大型文件（>2000 行）里的"小复制"是技术债的种子。每隔一段时间扫描这些文件，找出"两边都写了同一段 30 行的逻辑"提取出来——`capacityWake.ts` 就是这种提取的产物。

### 一个真实的 race condition

capacityWake 的代码看起来简单，但隐含了一个微妙的 race condition 处理。看这段（`src/bridge/capacityWake.ts:35-44`）：

```typescript
function signal(): CapacitySignal {
  const merged = new AbortController()
  const abort = (): void => merged.abort()
  if (outerSignal.aborted || wakeController.signal.aborted) {
    merged.abort()
    return { signal: merged.signal, cleanup: () => {} }
  }
```

为什么要在 `addEventListener` 之前先**检查一次** `aborted`？因为 AbortSignal 的语义是"已经 aborted 的 signal，addEventListener 不会触发回调"——如果在 sleep 之前的瞬间 wake() 已经被调，但 signal() 还没 register 监听，那这次 sleep 会**永远等不到事件**。

预先检查相当于：

```
读取 atomic snapshot
  ↓
如果已 aborted → 直接返回 aborted 的 merged signal
否则 → register 监听器
```

这个 pattern 在并发编程里叫 **double-check pattern**——先 cheap check 一次，过了再做昂贵的事（这里是 register 监听）。少了第一次检查，会有一个非常窄但真实的 race window。

![Remote 4 文件 + Bridge 30 文件的依赖关系图：Remote 在底层、Bridge 在上层，箭头标注谁调用谁](images/ch19/02-img02.png)

### 新 controller 的"原子替换"

```typescript
function wake(): void {
  wakeController.abort()
  wakeController = new AbortController()
}
```

这两行的顺序很重要：**先 abort 旧的，再换成新的**。如果反过来：

```typescript
// 错误顺序示例
wakeController = new AbortController()  // ← 旧的还没 abort，监听者收不到通知
wakeController.abort()                   // ← abort 的是新的，没监听者
```

旧的 listeners 会永远等下去（因为它们绑的是旧 wakeController.signal）。

但即使顺序对了，还有一个并发风险：abort 旧的之后、换新的之前，如果有别的代码读到 `wakeController.signal`，它会拿到一个**已经 abort 的 signal**。这个时候去 sleep 会立刻醒——但醒之后 loop 会重新检查 atCapacity，仍然 at capacity 就再 signal()，这次拿到新 controller，正常工作。所以即使有这个窄窗口，最坏情况只是多 spin 一次循环，不会死锁。

JS 是单线程的，这种细节比 multi-threaded language（C++/Java）简单——不用考虑 memory ordering、happens-before 这些。但即使在 single-threaded JS 里，**事件回调的执行时序**仍然是个能踩坑的地方。

---

## 19.7 Remote 系统（src/remote/，1,127 行）

如果说 Bridge 解决"Web 客户端 ↔ 本机 Agent"的协作，那 Remote 解决的是更宽泛的问题：**任何远端 Anthropic Server 上的会话 ↔ 本机 Agent**——包括 Mobile App、Web、和未来可能加入的客户端。

### Remote 的 4 个文件

`src/remote/` 总共 4 个文件、1,127 行（数据见 `docs/canonical-numbers.md` 第 22 项）：

| 文件 | 行数 | 职责 |
|---|---|---|
| `RemoteSessionManager.ts` | 343 | 远程会话管理器，对接 Agent SDK |
| `SessionsWebSocket.ts` | 404 | WebSocket 客户端，连 `/v1/sessions/ws/{id}/subscribe` |
| `remotePermissionBridge.ts` | 78 | 把远端的权限请求桥接到本机 |
| `sdkMessageAdapter.ts` | 302 | SDKMessage 适配（远端格式 ↔ 本机 query loop 格式）|

合计 1,127 行——一个相对小但极关键的子系统。

### Remote 与 Bridge 的关系

容易混淆：Remote 不是 Bridge 的替代品，而是 Bridge **依赖**的底层组件。简化看：

```
[Web 客户端]
      ↕
[Anthropic Server: /v1/sessions/...]
      ↕  ◄──── SessionsWebSocket（remote/）
[RemoteSessionManager（remote/）] 拼接 SDKMessage 流
      ↕
[bridgeMain.ts 主循环] 把 SDKMessage 喂给 Agent SDK
      ↕
[Agent 内核] 跑 query / 调工具 / 流式输出
```

Bridge 是更高层的"业务编排"（"会话管理"、"轮询节奏"、"flushGate"、"trustedDevice"），Remote 是更底层的"传输适配"（WebSocket、消息格式翻译）。

### SessionsWebSocket：长连接的细节

`SessionsWebSocket.ts:36-50` 的常量值得读一读：

```typescript
// src/remote/SessionsWebSocket.ts:18-36
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
const PING_INTERVAL_MS = 30000

const MAX_SESSION_NOT_FOUND_RETRIES = 3
```

这些数字背后是好多个生产经验：

- **RECONNECT_DELAY_MS = 2000**：网络抖动后等 2 秒再重连——既不太短（避免在网络真坏时疯狂打 Server）也不太长（用户等不了）。
- **MAX_RECONNECT_ATTEMPTS = 5**：5 次还连不上就放弃——通常意味着 token 过期、session 已删、或网络长时间断。
- **PING_INTERVAL_MS = 30000**：每 30 秒发一次心跳——和大多数 NAT/Proxy 的连接超时（一般 60 秒）匹配，确保 NAT 表项不会因为 idle 被回收。
- **MAX_SESSION_NOT_FOUND_RETRIES = 3**：4001 close code（session not found）特殊处理，重试 3 次。原因 doc comment 说得很清楚：

> Maximum retries for 4001 (session not found). During compaction the server may briefly consider the session stale; a short retry window lets the client recover without giving up permanently.

意思是：**Server 在做 session 压缩（compaction）时，会有短暂窗口期让 session 看起来"不存在"**——这时候直接放弃是错的，应该等几秒再试。

### 协议：第一条消息必须是 auth

`SessionsWebSocket.ts:80-85` 的注释解释了协议：

```
Protocol:
1. Connect to wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe?organization_uuid=...
2. Send auth message: { type: 'auth', credential: { type: 'oauth', token: '...' } }
3. Receive SDKMessage stream from the session
```

WebSocket 握手时不带 token（query string 不安全），握手成功后第一条 application 消息才发 auth。这是大多数生产 WebSocket 的做法——比起把 token 放 query string 或 Authorization header，"协议第一帧"更不容易被中间层日志记下来。

### 心跳与重连

`SessionsWebSocket` 同时承担三件事：

1. **保活**：每 30s 发 ping，对方有 pong 回来才算活；
2. **重连**：网络抖动后用指数退避（虽然源码里是固定 2000ms，但能看到 `reconnectAttempts` 计数器，意味着上限到了 5 次会停）；
3. **状态机**：`closed`、`connecting`、`open`、`closing`、`reconnecting` 几个状态——`onClose` 只在永久关闭时触发，`onReconnecting` 在临时关闭时触发。

注释里有一句很 telling 的话：

> A hardcoded allowlist here would silently drop new message types the backend starts sending before the client is updated.

这是关于 forward compatibility 的：新的消息类型上线时，如果 client 写死了 allowlist，会把新消息丢掉而不知道。所以这里**只检查"是不是 string 类型的 type 字段"**——具体值交给下游处理。Server 加新类型时 client 自动透传，不需要发版同步。

### sdkMessageAdapter 与 remotePermissionBridge

`sdkMessageAdapter.ts`（302 行）做的事和 `inboundMessages.ts` 类似——把远端格式（可能带 `mediaType` 这种 camelCase）翻译成 Agent 内核能直接吃的 SDKMessage。但它处理的不只是 user message，也处理 assistant 流（thinking、tool_use、tool_result 块）。

`remotePermissionBridge.ts`（78 行）只做一件事：**把远端的"权限请求"（"AI 想跑这个 bash 命令，要不要批"）转换成本机权限系统认识的格式，转回去时再翻译回远端格式。**

权限系统的两侧都有：

- 远端：Web/Mobile UI 弹一个 "AI wants to run `rm -rf node_modules`. Allow?" 对话框；
- 本地：`src/permissions/` 的 5 种权限模式 + 持久化的 `dontAsk` 列表。

`remotePermissionBridge.ts` 做的就是这两边之间的 RPC——本地决策（"用户在远端点了 Allow"）需要影响远端 UI 的状态变化、本地的 `dontAsk` 缓存。这种"权限决策的远程同步"在 Web/Mobile 模式里非常关键，否则用户点了一次 "Don't ask again" 之后 Bridge 不知道，下次又会弹。

![Voice 录音 + STT + keyterms 的数据流图：麦克风 → PCM → WebSocket → Server STT → 转录文本 + keyterms → 填回 REPL](images/ch19/04-img04.png)

### Remote 模块的"独立可复用"性

读 Remote 4 个文件你会发现一个有意思的事实：**Remote 模块依赖的东西很少**——基本只依赖：

- `entrypoints/agentSdkTypes`（SDKMessage 类型）；
- `entrypoints/sdk/controlTypes`（控制消息类型）；
- `utils/teleport/api`（Teleport HTTP API client）；
- `utils/debug` / `utils/log`（日志）；
- `utils/mtls` / `utils/proxy`（TLS / 代理工具）。

它**没有**依赖：

- React / Ink（不知道 UI）；
- permissions（不直接做权限决策，靠 callback）；
- bridge（反过来——Bridge 依赖 Remote）；
- agent core（不直接调 query loop）。

这种"低依赖"让 Remote 可以**独立移植**——理论上你可以把这 4 个文件抽出来用在另一个 SDK 里，只要那个 SDK 也有 SDKMessage 抽象。这是一种很好的层次划分方式。

### 客户端版本兼容的小工程

`SessionsWebSocket.ts:8-15` 提到了一个版本兼容细节：

```typescript
function isSessionsMessage(value: unknown): value is SessionsMessage {
  if (typeof value !== 'object' || value === null || !('type' in value)) {
    return false
  }
  // Accept any message with a string `type` field. Downstream handlers
  // (sdkMessageAdapter, RemoteSessionManager) decide what to do with
  // unknown types. A hardcoded allowlist here would silently drop new
  // message types the backend starts sending before the client is updated.
  return typeof value.type === 'string'
}
```

这个 type guard 故意做得**最宽松**——只要是对象 + 有 type 字段就放过。原因是 forward compatibility：Server 加新消息类型时不用等 client 升级。

但这种"最宽松的入口"有个隐患：拼写错误的 type 也会被放过。`type: 'mssage'` （拼错）会被透传到下游，下游也没有这个类型的 handler，最终就**默默被丢掉**。怎么发现这种 bug？一般靠：

1. **Server 端验证**：Server 在发出之前 schema 校验自己的消息；
2. **Client 端 telemetry**：未识别的 type 上报 analytics；
3. **集成测试**：跨 client/server 版本的端到端测试。

光靠 client 端的 type guard 是不够的——这是 SOA 系统跨版本协作的现实。

---

## 19.8 Server 系统（src/server/）

`src/server/` 是另一个**容易和 Bridge / Remote 混淆**的目录。它的名字叫 server，但**不是"Anthropic Server"**——它是 Claude Code CLI 自己起的一个**本地小 Server**，用来做"Direct Connect"模式。

### Direct Connect 是什么

3 个文件，358 行：

| 文件 | 行数 | 职责 |
|---|---|---|
| `createDirectConnectSession.ts` | 88 | 在远端 server 上创建一个 session（HTTP POST）|
| `directConnectManager.ts` | 213 | DirectConnect 会话管理器（WebSocket + 消息路由）|
| `types.ts` | 57 | 类型定义（Connect 请求/响应 schema）|

Direct Connect 的本质是：**绕过 Anthropic Server，直接连一个用户自己跑的 server**。

```
正常 Bridge 模式:
  本机 CLI ←→ Anthropic Server ←→ Web 客户端

Direct Connect 模式:
  本机 CLI ←→ 用户自己跑的 server（公司内网/自部署）←→ 客户端
```

为什么需要这种模式？

1. **企业合规**：某些大企业不允许员工的代码经由 Anthropic Server 转发——他们要求所有数据流量经过自己内网的 server。
2. **离线开发**：飞机上、火车上、防火墙后，连不到 Anthropic Server 但能连内网。
3. **第三方集成**：某些 IDE 插件想直接连 CLI，不经过任何中间层。

### createDirectConnectSession：HTTP POST 创建会话

`createDirectConnectSession.ts:9-15` 定义了错误类型：

```typescript
export class DirectConnectError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'DirectConnectError'
  }
}
```

接下来的逻辑是一个直白的 HTTP POST：

1. 拿 `DirectConnectConfig`（serverUrl + 可选 authToken）；
2. POST 到 `{serverUrl}/sessions`；
3. 用 `connectResponseSchema`（Zod schema）校验响应；
4. 解出 `sessionId` + `wsUrl`，返回。

这个文件只有 88 行——它故意**保持最小**。原因是 `directConnectManager.ts` 才是真正复杂的部分。

### directConnectManager：WebSocket + 消息路由

`directConnectManager.ts:42-54` 的 `DirectConnectSessionManager` 类是核心：

```typescript
export class DirectConnectSessionManager {
  private ws: WebSocket | null = null
  private config: DirectConnectConfig
  private callbacks: DirectConnectCallbacks

  connect(): void {
    const headers: Record<string, string> = {}
    if (this.config.authToken) {
      headers['authorization'] = `Bearer ${this.config.authToken}`
    }
    this.ws = new WebSocket(this.config.wsUrl, {
      headers,
    } as unknown as string[])
```

这里和 `SessionsWebSocket.ts` 不同——auth 通过 header 而不是协议第一帧。原因是 Direct Connect 假设连的是用户自己的 server，可以把 token 放 header（而 Anthropic Server 是公网的，不放 header）。

消息处理（`directConnectManager.ts:62-78`）：

```typescript
this.ws.addEventListener('message', event => {
  const data = typeof event.data === 'string' ? event.data : ''
  const lines = data.split('\n').filter((l: string) => l.trim())
  for (const line of lines) {
    let raw: unknown
    try {
      raw = jsonParse(line)
    } catch {
      continue  // 解不开就跳过，不让一个坏帧搞挂整个连接
    }
    if (!isStdoutMessage(raw)) {
      continue
    }
    // ... 路由到 callback
  }
})
```

注意几点工程性细节：

1. **NDJSON 协议**：消息是用换行分隔的 JSON（每行一个独立 JSON 对象）——这是 streaming 协议的常见做法，比纯 JSON 数组好——能流式处理。
2. **容错**：解 JSON 失败 `continue`，不抛异常——一个坏帧不影响后续帧。
3. **类型守卫**：`isStdoutMessage` 是一个简单的 type guard，只校验"是不是对象 + 有 `type` 字段"，具体类型交给下游分发。

### Direct Connect 的设计哲学

读完 358 行 server 代码会发现：**它故意做得不像 Bridge**。

- Bridge 用 HTTP 长轮询 + JWT + trustedDevice + flushGate + capacityWake；
- Direct Connect 只用 WebSocket + 简单 Bearer token。

为什么？因为 Direct Connect 的客户群不一样：

- Bridge 客户是公网用户，Server 是 Anthropic 自己跑的，要面对各种攻击和 abuse；
- Direct Connect 客户是企业 IT，自己跑 server，面对的是受信任内网。

所以 Direct Connect 把"加密、信任、配额"全部交给企业自己处理（用他们自己的 mTLS / VPN / WAF），CLI 只负责最简单的 WebSocket 通讯。**复杂度被推到正确的地方**——这是 358 行 vs 12,613 行的根本原因。

![Voice 录音多后端切换决策树：CoreAudio (mac) / PulseAudio (Linux) / SoX rec / arecord，每个分支带兜底条件](images/ch19/05-img05.png)

### types.ts 的 schema 设计

`src/server/types.ts` 只有 57 行，用 Zod 定义了 Direct Connect 的两个核心 schema：

- `connectRequestSchema`：发起 Connect 的请求体（session 期望、metadata）；
- `connectResponseSchema`：Server 返回的响应（sessionId、wsUrl、可能的 error）。

这种 Zod schema 的好处不是"验证数据"——更重要的是 **生成 TypeScript 类型 + 在协议变更时强制更新调用方**。如果 server 加了一个必填字段，client 这边的 `parse(...)` 会立刻在类型层面报错，不需要等运行时挂掉。

### Direct Connect 的应用场景

谁会真的用 Direct Connect？典型客户：

1. **大型金融企业**：合规要求"代码不能出企业 VPC"——他们部署一个内部 Claude Code Server，员工的 Claude Code CLI 连这个 Server，不连 Anthropic Server。
2. **离线开发环境**：飞机/船只/野外作业站——网络条件差，Anthropic Server 不可达，但内网有缓存的模型。
3. **第三方 Agent 框架**：某些公司想用 Claude Code 作为 IDE 后端，他们用 Direct Connect 把 IDE 直接连本机 Claude Code。

CLI 这一侧 358 行就够了——因为这些场景共享同一个简单需求："连一个用户指定的 server，开 WebSocket，传 SDKMessage"。

---

## 19.9 Voice 系统

Claude Code 支持**语音输入**——按住一个键说话，松开发送转录文本。这功能在 `claude.ai` (Anthropic 官方版本) 启用，第三方 fork 不一定有（被 `feature('VOICE_MODE')` gate 住）。

### Voice 系统 1,229 行的全景

数据见 `docs/canonical-numbers.md` 第 20 项：voice 总 LOC = **1,229**，分布在多个目录：

| 文件 | 行数 | 职责 |
|---|---|---|
| `src/services/voice.ts` | 525 | 录音服务（CoreAudio / SoX / arecord 后端切换）|
| `src/services/voiceStreamSTT.ts` | 544 | STT WebSocket client（流式语音识别）|
| `src/services/voiceKeyterms.ts` | 106 | 关键词提示（提升专业术语识别率）|
| `src/voice/voiceModeEnabled.ts` | 54 | "voice mode 是否可用"判断 |

加上 hooks 和 commands 的相关文件（`useVoice.ts` 1,144 行、`useVoiceEnabled.ts` 25 行、`useVoiceIntegration.tsx`、`commands/voice/`），整个 voice 体系跨越 6+ 个文件——但核心还是 services 这 4 个。

### 录音：cpal 原生模块 + 三种后备

`voice.ts:6-12` 的注释把录音架构讲得明明白白：

> Recording uses native audio capture (cpal) on macOS, Linux, and Windows for in-process mic access. Falls back to SoX `rec` or arecord (ALSA) on Linux if the native module is unavailable.

平台栈是这样的：

```
macOS:  audio-capture-napi（CoreAudio.framework）
Linux:  audio-capture-napi（PulseAudio/PipeWire）+ fallback: arecord (ALSA) / SoX rec
Windows: audio-capture-napi（WASAPI）
```

为什么要这么多备份？因为 macOS 和 Windows 的"麦克风权限"机制不一样、Linux 各发行版的音频栈千差万别（PulseAudio / PipeWire / 纯 ALSA / WSL 没声卡），单一方案覆盖不了所有用户。

### 一个 cold start 噩梦

`voice.ts:14-20` 透露了一个有趣细节：

> audio-capture.node links against CoreAudio.framework + AudioUnit.framework; dlopen is synchronous and blocks the event loop for ~1s warm, up to ~8s on cold coreaudiod (post-wake, post-boot). Load happens on first voice keypress — no preload, because there's no way to make dlopen non-blocking and a startup freeze is worse than a first-press delay.

也就是说：

- 加载 audio-capture.node（一个 native binding）会**同步阻塞 event loop 1-8 秒**（Mac 上 coreaudiod 还没起来时尤甚）；
- 没法异步加载（dlopen 没有 async 版本）；
- 选择延迟到"用户首次按 voice 键"才加载——CLI 启动快，第一次说话慢一点。

这是工程上的**冷启动 vs 首次延迟**的经典权衡。Claude Code 选择优化冷启动，因为不是所有用户都用语音。

### voiceStreamSTT：WebSocket 流式 STT

`voiceStreamSTT.ts:1-12` 解释了协议：

> Anthropic voice_stream speech-to-text client for push-to-talk. Connects to Anthropic's voice_stream WebSocket endpoint using the same OAuth credentials as Claude Code. The endpoint uses conversation_engine backed models for speech-to-text.

工作流：

```
1. 用户按下 voice 键 → useVoice hook 触发录音开始
2. voice.ts 启动 audio-capture-napi → 拿到 PCM stream
3. voiceStreamSTT.ts 打开 WebSocket → wss://.../voice_stream
4. 用户说话 → PCM 帧 → WebSocket 二进制帧上行
5. Server 流式返回 TranscriptText（中间结果）+ TranscriptEndpoint（最终）
6. 用户松开键 → CloseStream 控制帧 → WebSocket 关
7. 最终 transcript 填进 REPL 输入框 → 用户按 Enter 发送
```

协议是 JSON 控制帧（KeepAlive / CloseStream）+ 二进制音频帧的混合 ——和大多数流式 STT 协议（如 Deepgram / Google Speech）一脉相承。

### voiceKeyterms：让"MCP"不再变成"empty pee"

`voiceKeyterms.ts` 解决一个看起来小但极烦人的问题：**编程术语会被通用 STT 模型误识别**。

举几个例子：

- "MCP" → "empty pee" / "I am a P"
- "regex" → "rejects" / "rejects"
- "subagent" → "sub agent" / "subagent" / "sub a gent"
- "TypeScript" → "type script" / "type's script"

`voiceKeyterms.ts` 的方案是把高频被误认的术语作为**关键词提示**（Deepgram-style "keywords"）传给 STT engine：

```typescript
// src/services/voiceKeyterms.ts:14-26
const GLOBAL_KEYTERMS: readonly string[] = [
  'MCP',
  'symlink',
  'grep',
  'regex',
  'localhost',
  'codebase',
  'TypeScript',
  'JSON',
  'OAuth',
  'webhook',
  'gRPC',
  'dotfiles',
  'subagent',
  'worktree',
]
```

并且——更聪明的——还会从**当前会话上下文**抽词：项目名、git branch 名、文件名（`splitIdentifier` 把 camelCase / kebab-case / snake_case 拆成单词）。例如你在一个名叫 "claude-code-sourcecodes" 的仓库下面 dev 一个 branch 叫 "feat/voice-keyterms"，这两个名字会被自动加入 keyterms。

doc comment 解释为什么不在列表里加 "Claude" / "Anthropic"：

> Note: "Claude" and "Anthropic" are already server-side base keyterms. Avoid terms nobody speaks aloud as-spelled (stdout → "standard out").

第二句很妙：**别加用户根本不会念那个拼写的词**——比如没人念 "stdout" 念三个音节 "ess-tee-dee-out"，他们念 "standard out"——所以加 "stdout" keyterm 反而帮倒忙。

![Teleport vs Dispatch 对比图：Teleport 是单 session 在设备间转移、Dispatch 是 fork 出独立 agent 在云端跑](images/ch19/07-img07.png)

### voiceModeEnabled：双 gate 机制

`voiceModeEnabled.ts:8-15` 显示 voice 有**两道 gate**：

```typescript
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

- **build-time gate**：`feature('VOICE_MODE')` ——只有 ant-build（Anthropic 官方）才编进 voice 代码，第三方 fork 没有。这个 gate 是 bun 的 bundle-time 替换，不是运行时 if，所以非 ant-build **完全没有** voice 代码。
- **runtime gate**：`tengu_amber_quartz_disabled` ——GrowthBook 的紧急停止开关。如果出事了 ops 可以一键 disable 所有用户的 voice。

加上 `hasVoiceAuth()`（OAuth token 必须存在），合计 3 道 gate 才能用 voice：编译 flag + GrowthBook + OAuth。

### `/voice` 命令

`commands/voice/voice.ts` 是用户的入口——`/voice` 命令切 voice 模式 on/off。代码逻辑：

1. 检查 `isVoiceModeEnabled()`（合上面三道 gate）→ 不通过给提示；
2. 切 `voiceEnabled` 在 `userSettings`；
3. 触发 `settingsChangeDetector` 让 UI 实时更新；
4. 打 analytics（`tengu_voice_toggled`）。

> **教学点**：Voice 系统是 Claude Code 里最"操作系统级"的子系统——它要面对 macOS / Linux / Windows 三个平台、native module 的 dlopen 阻塞、多种音频后端、流式 STT 协议、领域术语识别、紧急 kill switch。1,229 行能撑起这些是因为绝大部分复杂度被推到 audio-capture-napi（vendor）和 Anthropic STT Server 上——CLI 这边只做"录音 + 上传 + 显示"。

### probeArecord：探测 Linux 上的麦克风可用性

`voice.ts:71-90` 有一段非常细的工程逻辑：

> hasCommand() only checks PATH; on WSL1/Win10-WSL2/headless Linux the binary exists but fails at open() because there is no ALSA card and no PulseAudio server. On WSL2+WSLg (Win11), PulseAudio works via RDP pipes and arecord succeeds.

意思是：检查 `arecord` 在不在 PATH 里**不够**——它存在不代表能录音。WSL1 / 旧 WSL2 / 没显示器的 Linux 都"有 arecord 但开不了 device"。所以代码 actually spawn 一次 arecord 用真实参数试，等 150ms：

- 还活着 → device 开成功，可以用；
- 已退出 → 看 stderr 报什么错，记录日志。

这种"实际探测而不是宣传探测"的做法在跨平台软件里是必修课。光读 PATH / 检查 binary 存在 = 假阳性高。Memoize 探测结果防止每次按 voice 键都重复一遍。

![Mobile 集成架构：iOS/Android App ↔ Anthropic Server (WebSocket) ↔ 本机 Bridge ↔ Agent；旁边标注 push 通知是 Server 发 APNS/FCM](images/ch19/08-img08.png)

### useVoice hook：1,144 行的协调器

`src/hooks/useVoice.ts` 是 Voice 这一坨里的"中枢"，1,144 行——比 services/voice.ts 还大。它做的事：

1. **键盘按键监听**：监听 push-to-talk 按键（默认是 `Ctrl+V`，可以自定义）；
2. **录音生命周期**：按下→开始录音、松开→停止录音、用户切走 app→中断录音；
3. **STT WebSocket 协调**：和 `voiceStreamSTT.ts` 配合，把音频流发出去、把转录结果填进输入框；
4. **错误反馈**：录音失败、网络断、token 过期都要给用户合理提示；
5. **设置整合**：响应 `voiceEnabled` settings 变化、language 选择、keyterms 注入；
6. **平台差异**：mac 和 Linux 的按键事件、Windows 的麦克风权限差别。

1,144 行其实**不算多**——这种"在多个异步流之间协调"的代码很难压缩。每个用户故事（"录音中断怎么办"、"识别一半网断了怎么办"）都要明确响应。

### Voice 的隐私模型

Voice 上传的是用户的麦克风原始音频——这是非常敏感的数据。Anthropic 的策略：

1. **不持久化**：音频流式上传给 STT engine，转录完之后服务器不保存原始音频；
2. **OAuth 绑定**：必须有 Anthropic OAuth token，不接受 API key 路径；
3. **Voice mode 是 opt-in**：用户必须 `/voice` 显式启用，不会默认开；
4. **键控触发**：必须按住按键说话才录音，不会一直监听（不像 Hey Siri 这种 always-on 模式）。

这套约束的意思是：voice 用户清楚地知道"我什么时候在录音"——不会发生"忘了 mic 是开的"那种隐私事故。

---

## 19.10 SSH Session

如果你在云上跑代码（例如 EC2、GCP VM），你想在那台远程机器上跑 Claude Code，但又想用本机终端的 UI（你不想 SSH 进去之后再装一遍 Claude Code）——`claude ssh` 模式就是为这个设计的。

### useSSHSession：和 Direct Connect 同构

`src/hooks/useSSHSession.ts` 共 241 行，doc comment 写得很有意思（`useSSHSession.ts:1-9`）：

> REPL integration hook for `claude ssh` sessions. Sibling to useDirectConnect — same shape (isRemoteMode/sendMessage/cancelRequest/disconnect), same REPL wiring, but drives an SSH child process instead of a WebSocket. Kept separate rather than generalizing useDirectConnect because the lifecycle differs: the ssh process and auth proxy are created BEFORE this hook runs (during startup, in main.tsx) and handed in; useDirectConnect creates its WebSocket inside the effect.

要点：

1. **接口一致**：`useSSHSession` 和 `useDirectConnect` 都暴露 `{ isRemoteMode, sendMessage, cancelRequest, disconnect }` 同样的接口——所以 REPL 能用同样的代码处理两种模式。
2. **生命周期不同**：但实现没合并，因为：
   - `useDirectConnect` 在 effect 里创建 WebSocket；
   - `useSSHSession` 在 main.tsx 启动早期就创建 ssh 子进程 + 一个 auth proxy；hook 只是接管已经在跑的进程。
3. **分开 vs 统一**：选择"分开"（不强行抽公共抽象）的理由是真实的——这两边的"创建时机"差异巨大，硬合在一起会让 useDirectConnect 变得很丑。

### claude ssh 的工作原理

```
本机:                                远程:
  用户终端                              远程服务器
       │                                  │
   claude ssh user@host ◄────────────►  /usr/bin/sshd
       │                                  │
       ▼                                  ▼
   ssh 子进程 (本机) ◄═══════════════►  ssh server
       │                                  │
       ▼                                  ▼
   useSSHSession hook              远程 claude code 进程
   （绑定到 stdin/stdout）        （在远程 host 上跑 Agent）
       │
       ▼
   本机 React + Ink REPL UI（用户看到的）
```

关键点：

- **UI 在本机**——所以你看到的依然是熟悉的 React + Ink 渲染；
- **Agent 循环在远程**——所有工具调用（BashTool 跑的就是远程的 bash）发生在 EC2 上；
- **协议层是 stdin/stdout**——不需要 WebSocket，不需要 Anthropic Server——只用 ssh 自己的 streaming 通道。

这个模式的妙处是：**它让 Claude Code 变成"远程终端的本地 GUI"**。你的 ssh 私钥仍然在本机、远程的代码仍然在远程；中间没有 Anthropic Server——纯本地↔ssh 通讯。

### auth proxy：解决 OAuth 不能跨主机的问题

但有一个问题：模型推理需要 Anthropic API token。token 在本机的 `~/.config/claude` 里，远程 EC2 上没有。怎么办？

`useSSHSession` 的注释提到了 "auth proxy" ——这是 Claude Code 启动时另起的一个**本地 HTTP proxy**。原理：

1. 本机起一个 HTTP proxy server，监听某个本地端口；
2. ssh 加 reverse port forward：`ssh -R 8888:localhost:8888 user@host`；
3. 远程 claude code 把 `ANTHROPIC_API_BASE` 设到 `http://localhost:8888`；
4. 远程发 model 请求 → 走 ssh tunnel 回到本机 proxy → proxy 加上本机的 OAuth token → 转发到 `api.anthropic.com`；
5. 响应原路返回到远程。

这样**远程代码不需要本机 token**——本机 proxy 充当中间人帮它签 token。安全模型是"信任 ssh 通道 = 信任那台远程主机"，攻击者除非已经控制远程主机，否则拿不到本机 token。

### useTeleportResume：回到一个之前的 session

`src/hooks/useTeleportResume.tsx` 84 行，是另一个相关 hook——把"远程 session"和"本地 session"做"接力"。Teleport 的具体细节我们下一节讲。

> **教学点**：`claude ssh` 是 Claude Code 系统设计的一个隐藏闪光点——它把"UI 在本地、Agent 在远程"这个奇怪的拓扑做得近乎透明。241 行 hook + 一个 auth proxy 就达到了这种透明度，背后是对 ssh 协议、reverse port forward、auth flow 的扎实理解。

![![Teleport vs Dispatch 对比图：Teleport 是单 session 在设备间转移、Dispatch 是 fork 出独立 agent 在云端跑](images/ch19/02-img02.png)](images/ch19/09-img09.png)

### useSSHSession 与 useDirectConnect 的"接口同形"

回顾 `useSSHSession.ts` 的注释：

> Sibling to useDirectConnect — same shape (isRemoteMode/sendMessage/cancelRequest/disconnect), same REPL wiring, but drives an SSH child process instead of a WebSocket.

这种"接口同形但实现不同"的模式在 Claude Code 里反复出现。REPL 主组件不用关心"我在和 WebSocket 还是 ssh stdin 讲话"——它只调用：

```typescript
interface RemoteSession {
  isRemoteMode: boolean
  sendMessage(msg: SDKMessage): void
  cancelRequest(uuid: UUID): void
  disconnect(): void
}
```

底下接什么传输都行。这就是**面向接口编程**在大型项目里的好处——加一个新的 transport 不需要改 REPL 任何代码。

### SSH stdin/stdout 协议的边角

ssh 的 stdin/stdout 不像 WebSocket 那样有清晰的消息边界——它就是一坨 bytes stream。所以 SSH 模式的协议必须自己定义"一条消息从哪里到哪里"。约定俗成的做法是 **NDJSON**（newline-delimited JSON）——每行一个独立 JSON 对象。这和 `directConnectManager.ts:65` 处理 WebSocket message 时按 `\n` split 是一样的逻辑：

```typescript
const lines = data.split('\n').filter((l: string) => l.trim())
for (const line of lines) {
  raw = jsonParse(line)
  // ...
}
```

NDJSON 在 stream 中是 partial parse-friendly 的——你可以从任何字节位置开始读，找到第一个完整 line 就 parse。

### auth proxy 的延迟权衡

auth proxy 的方案虽然优雅，但它给每个 model 请求加了**双倍 RTT**：

```
不走 proxy:    远程 → api.anthropic.com  RTT = X
走 proxy:    远程 → 本机 → api.anthropic.com  RTT = ssh latency + X
```

如果 ssh 是从 SF 连到东京 staging（延迟 100ms），又要 round-trip 一次，model 请求总 RTT 就是 100ms (ssh) + 100ms (api.anthropic.com) = 200ms 起步。这个延迟代价是真实的。

不过对于交互式开发来说，这个延迟一般还可以接受——人感知到的是 token 流式输出的速度，不是单次请求的总 RTT。Streaming 让首字节延迟 ≈ ssh latency + first-token latency，比纯 round-trip 计算友好得多。

---

## 19.11 Teleport / Dispatch 模式

Teleport 和 Dispatch 是两个看起来类似但语义不同的"会话切换"模式：

- **Teleport（传送）**：把当前会话**搬到**另一个设备/进程上继续。本端关闭，对端接管。
- **Dispatch（派发）**：把一个任务**派出去**给一个 remote agent，自己继续干别的。多个会话并行存在。

### Teleport：会话的"接力"

`src/commands/teleport/index.js` 在公开版本里是个 stub：

```javascript
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

这是 build-time gating——`feature('TELEPORT')` 在公开 build 里关闭，所以 source map 和 import path 还在，但功能 disabled。完整的 Teleport 实现走 `src/utils/teleport/api.ts`：

```
src/utils/teleport/
├── api.ts                 — Teleport HTTP API client
├── environmentSelection.ts — 选择目标环境
├── environments.ts         — environment 类型与列表
└── gitBundle.ts            — 把 git 工作区打成 bundle 一起传送
```

### Teleport 的真实场景

Teleport 解决的是"你的笔记本电脑要没电了，但 Claude Code 正在跑一个长任务，你不想停"。流程：

```
1. 在笔记本上：用户打 /teleport
2. CLI 把当前 session 上下文（消息历史 + 工作区状态）打包：
   - SDKMessage 数组（对话）
   - git working copy（gitBundle.ts 把 uncommitted 改动打成 bundle）
   - 当前环境变量（白名单过滤）
3. 上传到 Anthropic Server，注册一个 "remote environment"
4. Server 返回一个二维码 / 短链
5. 用户在另一台机器（台式机/工作站）上打 claude，输入 /teleport-resume
6. 那台机器从 Server 拉 session 上下文 + git bundle
7. 解 git bundle 到本地（保留 uncommitted 改动）
8. 接着上次对话继续——历史消息全在
```

### useTeleportResume hook

`src/hooks/useTeleportResume.tsx` 的代码（编译后版本）：

```javascript
// src/hooks/useTeleportResume.tsx:14-25
export function useTeleportResume(source) {
  const $ = _c(8);
  const [isResuming, setIsResuming] = useState(false);
  const [error, setError] = useState(null);
  const [selectedSession, setSelectedSession] = useState(null);
  // ...
  t0 = async session => {
      setIsResuming(true);
      setError(null);
      setSelectedSession(session);
```

注意 `_c(8)` 是 React Compiler 优化产物——React Compiler 把 useMemo / useCallback 自动加上去，这里 `8` 是 memo cache slot 数。这种代码在编译后看起来怪，但运行时性能比手写的 hooks 好。

### Teleport 的两种来源

`useTeleportResume.tsx:13` 定义了 source 类型：

```typescript
export type TeleportSource = 'cliArg' | 'localCommand';
```

- **cliArg**：用户启动 CLI 时传 `--teleport-resume <id>`；
- **localCommand**：用户在 REPL 里打 `/teleport-resume`。

两者用同一个 hook，因为后续处理一致——只是触发时机不同。

### Dispatch / Schedule Remote Agents

Dispatch 走的是一条不同的路线——`src/skills/bundled/scheduleRemoteAgents.ts` 是 17 个 bundled skills 之一。它做的是：

```
1. 用户在 Claude Code 里说："起一个 remote agent，每周一上午跑这个 cleanup 脚本"
2. SkillTool 调用 scheduleRemoteAgents skill
3. skill 调 RemoteAgent task type（参考 docs/canonical-numbers.md 第 10 项的 7 种 Task 类型）
4. 在 Anthropic Server 上注册一个 cron-driven agent
5. Server 按 cron 触发该 agent，独立运行
```

和 Teleport 不同：Dispatch 创建的是**独立的、和当前 session 没有继承关系的**新 agent。当前 session 继续干自己的事。

### `/schedule` 命令对应

用户层面，对应的是 superpowers 系列的 `/schedule` command（见 user CLAUDE.md 的"周期性任务/轮询"那行）。Claude Code 内置的 `anthropic-skills:schedule` skill 触发 RemoteAgent 创建——本质上 `/schedule` 是 Dispatch 的一个高层封装。

### Teleport vs Dispatch 速查

| 维度 | Teleport | Dispatch |
|---|---|---|
| 当前 session | 暂停/转移 | 继续运行 |
| 对端 session | 继承当前历史 | 独立新建 |
| 触发 | 主动（用户打命令）| 主动 |
| 持续时间 | 一次性 | 可定时（cron）|
| 典型场景 | 切换设备 | 后台任务 / 监控 |
| 数据传输 | git bundle + 消息历史 | 任务描述 + 必要 context |

> **教学点**：Teleport 和 Dispatch 共用 `src/utils/teleport/api.ts` 这个 API 层（注释里 `RemoteMessageContent` 在两边都用），但产品形态完全不同。这种"底层共享 API、上层语义分裂"是 Anthropic 这类工具型产品的常见做法——一个 RPC 通道服务多个用户用例。

![![跨设备工作流时间轴：08:30 桌面→10:00 手机→14:00 SSH→18:00 Dispatch，箭头标注用了哪些子系统](images/ch19/05-img05.png)](images/ch19/04-img04.png)

### gitBundle：把工作区一起带走

`src/utils/teleport/gitBundle.ts` 处理 Teleport 的一个关键技术问题：**怎么把"还没 commit 的工作区"带到对端**。

简单方案：直接把 workspace 文件全部 tar 起来上传。但这有问题：

- node_modules / 编译产物会让 tar 巨大；
- gitignored 文件应该被排除（用户的 .env 不该上传）；
- 二进制文件浪费带宽。

更好的方案是**用 git bundle**：

```
1. git bundle create /tmp/snap.bundle --all
   （把当前所有 commit 打成一个 bundle）
2. git diff HEAD --binary > /tmp/uncommitted.patch
   （把 uncommitted 改动单独存成 patch）
3. 上传 bundle + patch
4. 对端：
   git clone /tmp/snap.bundle <project>
   git apply /tmp/uncommitted.patch
   （workspace 完整复刻）
```

这样：

- 历史 commit 走 git bundle 的高效格式（增量、压缩）；
- uncommitted 改动走 patch（精确还原 working tree）；
- gitignored 文件自动跳过；
- 二进制文件 git 会自动 dedupe。

整个 transport 体量可能是直接 tar 的 1/10。

### environment 选择

`src/utils/teleport/environments.ts` 定义了"environment"概念：

```
environment = 一台远程机器上预备好的环境，包括：
  - 已安装的 toolchain（node、python、rust）
  - 已克隆的常用仓库
  - 已配置的环境变量
```

Teleport 时用户选择把 session 转移到哪个 environment——可以是另一台桌面、笔记本，也可以是 Anthropic 提供的 cloud environment（CCR）。

`environmentSelection.ts` 是这个选择的 UI 逻辑——列出可用 environments、显示连接状态、让用户选一个。

### Dispatch 的典型用法

`scheduleRemoteAgents` skill 暴露给上层使用的 prompt 大概是这样：

```
"在每周一上午 9 点跑 ./scripts/cleanup-old-branches.sh，
 如果有任何 branch 被删掉，给我发 Slack 通知"
```

skill 调用流程：

1. Claude（在你当前 session 里）调 SkillTool；
2. SkillTool 调用 scheduleRemoteAgents skill；
3. skill 里调 RemoteAgent task type；
4. task 在 Server 上注册一个 cron 触发器；
5. 每周一 09:00 Server 启一个 RemoteAgent 实例；
6. 这个 agent 自己跑 query loop、调 BashTool、调 SlackTool；
7. 跑完销毁，结果（如有 Slack 通知）独立投递。

这种"派出去的 agent"是一种**真正的并行**——和 Teleport 不一样，原 session 没有受到任何影响。

---

## 19.12 Mobile 集成

Claude 还有 iOS / Android Mobile App。Mobile App 不能在手机上跑 Claude Code 进程（手机连 git / shell / file system 都没法搞），所以它的工作模式是 **Web 模式的变体**：手机当 UI，连本机 Bridge 跑代码。

### `/mobile` 命令：扫码下载 + 配对

`src/commands/mobile/mobile.tsx` 是用户启动 Mobile 集成的入口。它做的事很简单：

```typescript
// src/commands/mobile/mobile.tsx:13-22
const PLATFORMS: Record<Platform, {
  url: string;
}> = {
  ios: {
    url: 'https://apps.apple.com/app/claude-by-anthropic/id6473753684'
  },
  android: {
    url: 'https://play.google.com/store/apps/details?id=com.anthropic.claude'
  }
};
```

`/mobile` 显示一个二维码——扫码下载 Mobile App。用户装好 App 登录之后，App 会自动扫描"用户名下当前在线的 Bridge 设备"，列出来选一台连。

### `/desktop` 命令：DesktopHandoff

`src/commands/desktop/desktop.tsx` 是 mobile 的对应物——把当前 mobile session "handoff" 到桌面：

```typescript
// src/commands/desktop/desktop.tsx
import { DesktopHandoff } from '../../components/DesktopHandoff.js';
export async function call(onDone) {
  return <DesktopHandoff onDone={onDone} />;
}
```

实现都在 `DesktopHandoff` 组件里。Handoff 的语义和 Teleport 类似——把当前 session 挪到桌面继续。区别在于 mobile/desktop 是**预绑定的设备对**（同一个 Anthropic 账号下的两台设备），不需要扫码、不需要选 environment——内部直接走 server 的 trusted device 列表。

### Mobile Bridge 协议的特殊性

Mobile App 用 native WebSocket 连 Anthropic Server（不像 Web 走 HTTP 长轮询），所以：

- **更低延迟**：Mobile 看到 assistant token 流式输出基本和 Web 同步；
- **更省电**：WebSocket 比长轮询好，App 进入后台时连接被系统挂起，不会唤醒 CPU；
- **camelCase 兼容性**：iOS / Android 原生开发用 camelCase 习惯——Bridge 端的 `inboundMessages.ts:12-15` 那段 mediaType 兼容代码就是为这个 case 准备的。

### Push 通知集成

Mobile 模式还有一个 Web/SSH 没有的能力：**push 通知**。当 Agent 跑完一个长任务（用户走神去刷别的），App 会推一条 "Your task is done" 通知。

实现层面，是 Bridge 在 Server 注册一个 webhook，session_end 事件触发 → Server 调 APNS / FCM 推到用户手机。CLI 这边只做了"声明这个 session 想要 push 通知"——具体推送逻辑在 Server。

### Mobile 集成的工程挑战

读 `src/commands/mobile/` 会发现它代码不多——核心复杂度全在 Mobile App 自己的代码里（不在 Claude Code 仓库）和 Anthropic Server 上。CLI 这边只负责：

1. 显示二维码 / 配对码（让用户认识这台设备）；
2. 从 inboundMessages 接 mobile 发的消息（已经被 Bridge 通用处理了）；
3. 给 mobile 发 assistant 响应（也是 Bridge 通用通道）；
4. 处理 mediaType / media_type 这类格式差异（已经在 inboundAttachments）。

这种"边界小、复用多"的设计意味着以后加新的 frontend（比如 VR 眼镜、车载系统）都不会让 CLI 代码膨胀——只要新 frontend 能讲 SDKMessage，Bridge 就吞得下。

> **教学点**：Mobile 集成是"小客户端 + 大公共基础"的范例。CLI 在 mobile 这块代码不到 200 行，是因为底层 Bridge / Remote 架构已经处理掉了所有"跨设备消息搬运"的细节。

![Mobile 集成架构：iOS/Android App ↔ Anthropic Server (WebSocket) ↔ 本机 Bridge ↔ Agent；旁边标注 push 通知是 Server 发 APNS/FCM](images/ch19/01-img01.png)

### Mobile App 的本机 token 不暴露

Mobile App **不会**拿到你本机的 OAuth token——它只拿一个 Server 颁发的 short-lived session token，只能用来 subscribe 这个 session 的消息流。这种设计意味着：

- 手机被偷不会泄露你的 Anthropic 账号；
- 手机离 root 也只能拿 session token，不能借此操控其他 session。

CLI 这边也不需要把 token 传给 Mobile——所有 mobile 流量都走 Anthropic Server，CLI 只看到来自 Server 的 SDKMessage 流，不知道是手机还是 Web 发的（也不需要知道）。

### handoff 的状态保留细节

`/desktop` 命令做 mobile→desktop handoff 时，状态保留范围：

| 范围 | 保留 | 不保留 |
|---|---|---|
| 消息历史 | ✅ | |
| 工具调用记录 | ✅ | |
| 当前 working directory | ✅ | |
| 未完成的 prompt input | ✅ | |
| 临时键盘焦点 / 滚动位置 | | ❌ |
| 弹窗 / 对话框状态 | | ❌（重置）|
| 录音中的 voice 状态 | | ❌（被取消）|

"重要业务状态保留、UI 临时状态丢弃"是这种设计的合理边界——用户期望对话历史无缝衔接，但不期望"手机上滚到第 10 屏，桌面也滚到第 10 屏"。

---

## 19.13 UpstreamProxy（上游代理转发）

`src/upstreamproxy/` 740 行——这是个非常特殊的子系统，**只在 CCR session 容器里激活**。CCR 是 Anthropic 内部的"远程沙箱代码运行"系统（"Containerized Code Runtime"）；用户在 Web/Mobile 里点"在云端运行这个 task"时，Anthropic Server 会启一个 CCR 容器，里面跑一个 Claude Code 实例帮用户执行——upstreamproxy 就是该容器里的网络出口管控。

### 740 行做什么

```
src/upstreamproxy/
├── upstreamproxy.ts  — 285 行：容器启动时的环境装配
└── relay.ts          — 455 行：CONNECT → WebSocket 隧道
```

`upstreamproxy.ts:1-22` 的 doc comment 完整描绘了职责：

> CCR upstreamproxy — container-side wiring.
>
> When running inside a CCR session container with upstreamproxy configured, this module:
>   1. Reads the session token from /run/ccr/session_token
>   2. Sets prctl(PR_SET_DUMPABLE, 0) to block same-UID ptrace of the heap
>   3. Downloads the upstreamproxy CA cert and concatenates it with the system bundle so curl/gh/python trust the MITM proxy
>   4. Starts a local CONNECT→WebSocket relay (see relay.ts)
>   5. Unlinks the token file (token stays heap-only; file is gone before the agent loop can see it, but only after the relay is confirmed up so a supervisor restart can retry)
>   6. Exposes HTTPS_PROXY / SSL_CERT_FILE env vars for all agent subprocesses

每一步都是被生产事故磨出来的：

- **第 2 步 prctl(PR_SET_DUMPABLE, 0)**：防止同 UID 进程 ptrace 这个进程的 heap，把 token 偷走。
- **第 3 步合并 CA bundle**：CCR 的 MITM proxy 用的是 CCR 自签 CA，curl / Python 默认不信。把 CCR CA 拼到 system bundle 后面，让 subprocess 都信任。
- **第 5 步 unlink token file**：token 读进 heap 之后，立刻删除磁盘上的文件——agent 跑起来之后看不到这个文件，但 supervisor 想重启时还能重建（在 relay 起来之前不删）。

### Why CONNECT-over-WebSocket

`relay.ts:1-18`:

```
WHY WebSocket and not raw CONNECT: CCR ingress is GKE L7 with path-prefix
routing; there's no connect_matcher in cdk-constructs. The session-ingress
tunnel (sessions/tunnel/v1alpha/tunnel.proto) already uses this pattern.

Protocol: bytes are wrapped in UpstreamProxyChunk protobuf messages
(`message UpstreamProxyChunk { bytes data = 1; }`) for compatibility with
gateway.NewWebSocketStreamAdapter on the server side.
```

这段技术含量很高：

- **GKE L7 + path-prefix routing**：Anthropic 的 ingress 是 Kubernetes Engine 的 L7 LB，按 URL path 路由——L4 tunnel（CONNECT method）跑不通。
- **复用已有协议**：session-ingress 模块的 tunnel.proto 已经有 WebSocket-based bytes-stream 协议，upstreamproxy 直接复用，不重新发明。
- **Protobuf 包裹**：每段 bytes 包成 `UpstreamProxyChunk` protobuf，保证 stream 边界清晰。

### 何时需要 upstream proxy

容易混淆：upstreamproxy **不是给普通用户用的**——普通用户的 Claude Code 不经过任何 proxy，直连 `api.anthropic.com`。upstreamproxy 只在两种场景出现：

1. **CCR 沙箱**：Anthropic Server 让 Claude Code 在云端执行用户任务时，把容器内所有 outbound HTTP 都 MITM，注入企业凭据（DD-API-KEY）、记录审计日志。
2. **企业内网**：某些企业用 CCR-on-prem，自己跑 CCR + upstreamproxy。

普通 CLI 启动时这个模块根本不会被激活——`isUpstreamProxyEnabled()` 检测 `/run/ccr/session_token` 是否存在，没有就 short-circuit 退出。

### "fail open" 设计

`upstreamproxy.ts:18`：

> Every step fails open: any error logs a warning and disables the proxy. A broken proxy setup must never break an otherwise-working session.

每一步都"失败时不影响主流程"——proxy 装不上就当没 proxy，session 继续跑。这是因为 proxy 是**增强**功能（审计、凭据注入），不是**必须**功能；如果它崩了把 session 也带崩，CCR 容器集体不可用，损失太大。

> **教学点**：upstreamproxy 740 行是企业级安全的微缩影——MITM CA 注入、PR_SET_DUMPABLE、token 内存隔离、protobuf-over-WebSocket 路由穿透——这些细节在普通 CLI 里看不到，但反映了 Claude Code 作为企业产品的成熟度。

![![系统总图：Server 上半部 + 本机 CLI 下半部，标注每个子系统的行数和职责](images/ch19/04-img04.png)](images/ch19/02-img02.png)

### NO_PROXY_LIST 的"三种写法"

`upstreamproxy.ts:40-50` 有一个让人会心一笑的细节：

```typescript
const NO_PROXY_LIST = [
  // ...
  // Anthropic API: no upstream route will ever match, and the MITM breaks
  // non-Bun runtimes (Python httpx/certifi doesn't trust the forged CA).
  // Three forms because NO_PROXY parsing differs across runtimes:
  //   *.anthropic.com  — Bun, curl, Go (glob match)
  //   .anthropic.com   — Python urllib/httpx (suffix match, strips leading dot)
  //   anthropic.com    — apex domain fallback
```

不同语言/runtime 解析 `NO_PROXY` 环境变量的语义是不一样的：

- Bun / curl / Go 看 `*.anthropic.com` 这种 glob；
- Python urllib / httpx 看 `.anthropic.com`（带前导点）；
- 一些 runtime 只看 `anthropic.com`（apex 域）。

为了让所有 subprocess 都正确把 `*.anthropic.com` 排除出 proxy，**三种写法都加进去**——这是一个非常工程的小妥协。理论上 NO_PROXY 应该有 RFC 规定，实际上没有，每个 runtime 自己实现自己的语义。

这种"接住所有 runtime 解析差异"的智慧，只能从生产事故里学到。

### 与 sandbox-shell-ccr.sh 的镜像

`upstreamproxy.ts` 注释提到这个 NO_PROXY 列表"mirrors airlock/scripts/sandbox-shell-ccr.sh"——意思是 CCR 这边的 shell 脚本也维护着同一份名单。两边必须保持一致——任何一边漏掉一个域名，那个域名就会被双重 NAT（一次 proxy 一次直连），结果是连接失败。

跨语言的常量同步是大型分布式系统的常见痛点。Anthropic 的处理是 source code 注释里互相引用——human-readable 但脆弱（哪天 .sh 改了 .ts 不知道）。更鲁棒的做法是把名单做成单独的文件（YAML / JSON），两边都读它，但 Anthropic 这里选择了简单。

---

## 19.14 集成示意：CC 如何在终端、Web、Mobile 间无缝切换

把前面 13 节的所有部件拼起来，让我们看一个真实的"跨设备工作流"是什么样：

### 场景：周一早上的代码会议

**08:30 - 在公司桌面**

```
你: 打开 Claude Code 终端
   $ claude
   [终端 UI 启动，进入 REPL]
你: "帮我看看 main 分支最近 5 个 commit 的 diff，找出可能的 regressioan 点"
   [Agent 跑]
   - GitTool 拿 5 个 commit
   - GrepTool 搜可疑 pattern
   - 输出 3 个潜在 regression
你: "针对第 2 个，加一个 unit test"
   [Agent 写测试 + 跑测试，绿]
```

到这里都是**终端模式**——没有 Bridge / Remote。

**10:00 - 出门去客户现场**

```
你: 打开手机 Claude App
你: 在 App 里看到桌面上还在跑的 session（trusted devices 列表）
   [App → Anthropic Server: 列出该用户在线 sessions]
你: 点"接管这个 session"
   [App → Server: subscribe 到 session]
   [Server 通过 SessionsWebSocket 把 session 状态推给 App]
   [App 渲染上面的对话历史]
你: 在路上手机上输入："那个测试再跑一次确认"
   [App → Server → 桌面 Bridge → Agent → BashTool 跑测试]
   [测试结果反向流回 App]
```

这一段是 **Mobile → Bridge** 模式。Agent 仍然在你公司桌面上跑（你的 git 仓库在那台机器上），手机只是 UI。

**14:00 - 客户现场要 demo，但需要在客户的 staging 环境**

```
你: 拿出笔记本，连客户 VPN
你: 在笔记本上 ssh 到 staging server
   $ ssh deploy@staging.client.com
你: 在 staging 上启动 claude
   $ claude
   [客户 staging 上跑 Claude Code]
你: 但你想用本机 OAuth token（你不能给客户 staging 装你的 token）
   [辅助方案：claude ssh deploy@staging.client.com（从你笔记本启动）]
   [本机 ssh tunnel + auth proxy 起来]
   [staging 上的 claude code 通过 tunnel 用你本机 token]
你: "在 staging 上面跑 demo 脚本"
   [Agent 在 staging 上跑 demo，结果显示在你笔记本终端上]
```

这是 **SSH 模式**——UI 在你笔记本，Agent 在 staging server。

**18:00 - demo 结束，回家路上**

```
你: 在地铁上，手机重新打开 Claude App
你: 看到当时 staging 那个 session
你: 输入："明天早上 9 点提醒我把 demo 时改的那个 hotfix 提 PR"
   [Skill: scheduleRemoteAgents]
   [一个 RemoteAgent task 被注册到 Anthropic Server cron]
   [明天 09:00 这个 agent 自动跑：检查 hotfix 还没 PR → 提 PR → 通知你]
```

这是 **Dispatch 模式**——派一个 agent 在云端按时间运行，和你当前 session 解耦。

### 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                     Anthropic Server                        │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│   │ Session A    │  │ Session B    │  │ Cron Agents  │     │
│   │ (公司桌面)    │  │ (笔记本+SSH) │  │ (Dispatch)   │     │
│   └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
        ▲ ▲                  ▲                     ▲
        │ │                  │                     │
   ┌────┘ │       ┌──────────┘                     │
   │      │       │                                │
   ▼      ▼       ▼                                ▼
┌──────┐ ┌──────┐ ┌──────────────┐            ┌─────────┐
│ 公司  │ │ 手机  │ │ 笔记本        │            │ 后台运行 │
│ 桌面  │ │ App  │ │ +SSH staging │            │ 不需要  │
│      │ │      │ │              │            │ 你在线  │
└──────┘ └──────┘ └──────────────┘            └─────────┘
 ↑          ↑               ↑                       ↑
 │          │               │                       │
Bridge   Bridge        SSH tunnel +              RemoteAgent
(本机)  (远程→本机)    auth proxy             on Anthropic Server
```

### Session 跨设备一致性的三大保证

要让上面这种"无缝切换"成立，背后有三个关键保证：

1. **Session ID 是稳定的全局标识**：从公司桌面到手机到笔记本，看到的是同一个 session ID。`bridgePointer.ts` 的 pointer 机制保证了**任何客户端都能从某个偏移量开始接收消息**——历史消息不重放，新消息不丢。

2. **trustedDevice 把"我"和"我的设备们"绑定**：你的手机、桌面、笔记本都是同一个 trustedDevice 列表里的设备。任意一台都能 subscribe 任意一个 session（前提是你的账号有权访问这个 session）。

3. **Frontend 完全可插拔**：终端 UI / Web UI / Mobile UI / SSH UI 都消费同样的 SDKMessage 流。Agent 内核根本不知道"我在跟谁讲话"——它只知道有一个 message stream 在那。Frontend 切换不需要 Agent 介入。

<!-- IMAGE: 跨设备工作流时间轴：08:30 桌面→10:00 手机→14:00 SSH→18:00 Dispatch，箭头标注用了哪些子系统 -->

### 这个无缝度的代价

读到这里，你会意识到 12,613 行 Bridge + 1,127 行 Remote + 1,229 行 Voice + 740 行 UpstreamProxy 不是浪费——它们是为了让"跨设备协作"这个看起来魔法的事情成为可能。

但工程角度也得诚实：**这套架构的复杂度门槛很高**。一个第三方 fork 想完整复刻 Bridge / Remote / Mobile / Teleport / Dispatch 这套，可能要花上比写 Agent 内核还多的精力。这也是为什么 Claude Code 公开的 fork 一般只复刻终端模式——Bridge 那一坨牵涉 Anthropic Server，单方面 fork 没意义。

### 系统总图：所有部件如何咬合

把整个第十九章串起来，可以画一张概览图：

```
┌──────────────────────────────────────────────────────────────────┐
│                       Anthropic 端（Server）                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Sessions API │  │ STT Engine   │  │ Cron Agents  │           │
│  │ JWT Issuer   │  │ voice_stream │  │ Dispatcher   │           │
│  │ Trusted Dev  │  └──────────────┘  └──────────────┘           │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
        ▲                       ▲                  ▲
        │ HTTPS / WebSocket     │ WebSocket        │
        │                       │                  │
┌──────────────────────────────────────────────────────────────────┐
│                       本机 Claude Code CLI                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │            Bridge 30 文件 / 12,613 行                      │  │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────┐              │  │
│  │  │ JWT auth │  │ flushGate│  │ pollConfig  │              │  │
│  │  │ trustDev │  │ inbound* │  │ capacityWake│              │  │
│  │  └──────────┘  └──────────┘  └─────────────┘              │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Remote 1,127 │  │ Server 358   │  │ UpstreamProxy 740    │   │
│  │ WebSocket   │  │ DirectConnect│  │ CCR-only             │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │ Voice 1,229  │  │ SSH (hooks)  │                             │
│  │ STT + record │  │ auth proxy   │                             │
│  └──────────────┘  └──────────────┘                             │
│                            │                                     │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Agent 内核（src/core/）                       │  │
│  │     query loop / tools / permissions / hooks               │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

每一个外层模块都是 Agent 内核的"frontend"——它们用不同的传输协议接入同一个 Agent。

<!-- IMAGE: 系统总图：Server 上半部 + 本机 CLI 下半部，标注每个子系统的行数和职责 -->

### 从代码量看子系统重要性

按行数排序：

```
12,613 — Bridge（Web/Mobile 协作）
 1,229 — Voice（语音输入）
 1,127 — Remote（WebSocket 传输）
   740 — UpstreamProxy（企业沙箱）
   358 — Server / Direct Connect（自部署）
   325 — SSH 相关 hooks（远程 host）
   84  — Teleport hooks（设备转移）
   ~50 — Mobile / Desktop UI 命令
```

看出层级：Bridge 是核心、Remote 和 Voice 各占一席、其他都是补充。理解 Claude Code 的"远程能力"，把 Bridge 这 12,613 行读懂就成功了 70%。

### 看不见的"传输无关"哲学

读完这一章，最重要的收获不是任何具体协议或文件，而是一个观念：

> **Agent 内核不应该知道"谁在和它讲话"。它应该面向一个抽象的 message stream 工作。**

这个观念让 Claude Code 能在终端、Web、Mobile、SSH 之间自由切换。如果 Agent 内核 hardcode 了"我在终端 React 渲染上跑"，就不可能后来扩展出 Web；如果 hardcode 了"我连 Anthropic Server"，就不可能扩展出 Direct Connect。

把这个观念推到极致，就能解释为什么 Claude Code 是个 SDK——`@anthropic-ai/claude-agent-sdk` 把 Agent 内核独立出来，让任何人能用任何 transport 包裹它。这是第二十章 Agent SDK 的内容。

---

## 动手实践

### 练习 1：找出你当前 session 是不是 Bridge 模式

**目标**：观察"有没有走 Bridge 模式"在 CLI 行为上的区别。

**步骤**：

1. 终端启动 `claude`，**不**打开 Web。这是纯终端模式。
2. 在 REPL 里随便发一条消息，观察终端 UI 渲染（你会看到 Ink 渲染的 spinner、token 计数）。
3. 退出 (`Ctrl+D`)。
4. 这次启动 `claude --bridge` 或者打开 `https://claude.ai/code` 创建 session（你需要登录）。
5. 终端这次会显示一条不同的状态——`bridgeStatusUtil.ts` 渲染的"Bridge connected"提示。在 Web 客户端发消息，注意你的本机终端**也能看到 Web 用户在干嘛**（这是 `bridgeUI.ts` 的功劳）。

**反思**：终端模式没有 Bridge / 不和 Anthropic Server 持续 poll；Web 模式下你的本机变成了 Anthropic Server 的"代理执行器"。

### 练习 2：模拟 flushGate 的语义

**目标**：用 50 行代码验证 `FlushGate` 的语义，避免被生产事故教育。

**步骤**：

1. 把 `src/bridge/flushGate.ts` 完整读一遍（71 行）。
2. 写一个 test：
   ```typescript
   const gate = new FlushGate<string>()
   gate.start()
   const enqueued1 = gate.enqueue("msg1")  // true
   const enqueued2 = gate.enqueue("msg2")  // true
   const drained = gate.end()
   // 期望 drained === ["msg1", "msg2"]
   const enqueued3 = gate.enqueue("msg3")  // false（gate 已经 end）
   ```
3. 再写一个 test 验证 `deactivate()` vs `drop()` 的差别：deactivate 后 pending 还在，drop 后 pending 被清空。

**反思**：5 个方法（start/enqueue/end/deactivate/drop）的状态机一定要画图——只看代码很容易把 deactivate 和 drop 搞混。

### 练习 3：研究 voice keyterms 对识别率的影响

**目标**：体会"领域词汇提示"对 STT 准确率的提升。

**步骤**：

1. （需要 ant-build 或 voice mode 已开）启用 `/voice`。
2. 在一个非编程项目目录下尝试说："use MCP to grep regex in subagent worktree"——看 STT 转录结果。
3. cd 到 `claude-code-sourcecodes` 仓库再启动 voice，重复同一句话。后者会把 "subagent" / "worktree" 加进 keyterms（因为是 GLOBAL_KEYTERMS）。但更明显的差异是项目特有词汇——例如在一个名叫 "octopus-prime" 的项目里说 "deploy octopus-prime"，识别率会显著高于通用项目。
4. 读 `src/services/voiceKeyterms.ts` 的 `splitIdentifier` 实现，理解它怎么从项目名 / branch 名抽词。

**反思**：通用 STT 模型对编程术语的误识别是真实痛点，keyterms 是个非常工程的解法——不需要训练专门的"编程 STT"模型，靠提示词让通用模型变好。

### 练习 4：用 capacityWake 写一个可重置 sleep

**目标**：理解 AbortSignal 的"可组合"特性。

**步骤**：

1. 不看 `src/bridge/capacityWake.ts` 源码，自己实现一个：

   ```typescript
   function createCapacityWake(outerSignal: AbortSignal): {
     signal(): AbortSignal
     wake(): void
   } { /* 你来实现 */ }
   ```

2. 用法：
   ```typescript
   const wake = createCapacityWake(processSignal)
   const sig = wake.signal()
   sleep(1000, sig).catch(() => console.log('woken or aborted'))
   // 在另一个 callback 里：
   wake.wake() // 应该让上面的 sleep 立刻 reject
   ```

3. 实现完之后对照 `capacityWake.ts`——你会发现自己很可能漏掉了：
   - 预先检查 `aborted` 的 race protection；
   - cleanup 函数（`removeEventListener`）；
   - 用 `let` 让 `wakeController` 可替换。

**反思**：56 行的代码看起来短，但每一行都有讲究——并发 primitive 自己写一遍才能体会到这种"看似简单实则刁钻"。

### 练习 5：看 SessionsWebSocket 如何处理 4001 错误

**目标**：理解"短暂期无效 vs 永久无效"的区分。

**步骤**：

1. 读 `src/remote/SessionsWebSocket.ts:18-36`：

   ```typescript
   const MAX_SESSION_NOT_FOUND_RETRIES = 3
   ```

2. 顺着 `MAX_SESSION_NOT_FOUND_RETRIES` 找到使用它的地方——是哪个 close code 触发了重试？

3. 思考：如果服务器 close code 4001 直接被当作"永久错误"处理，会发生什么？（提示：阅读注释里关于 "compaction" 的部分）

4. 设计：再加一类"短暂错误"码（比如 4002 = "服务器临时过载"）你会怎么实现？是加常量，还是引入一个分类函数？

**反思**：错误码的"永久 vs 短暂"分类是分布式系统设计的关键决策。读多个真实系统的 reconnect 逻辑能看到这种分类的不同流派。

---

## 源码对照表

| 教学概念 | 源码位置 | 关键行 |
|---|---|---|
| Bridge 总入口 | `src/bridge/bridgeMain.ts` | 全 2,999 行 |
| Bridge HTTP client | `src/bridge/bridgeApi.ts` | 1-50 |
| JWT 解码 | `src/bridge/jwtUtils.ts` | 21-32 |
| trustedDevice 注册 | `src/bridge/trustedDevice.ts` | 15-30 |
| 入站消息归一化 | `src/bridge/inboundMessages.ts` | 21-25 |
| 入站附件处理 | `src/bridge/inboundAttachments.ts` | 全 175 行 |
| FlushGate 状态机 | `src/bridge/flushGate.ts` | 16-71 |
| pollConfig schema | `src/bridge/pollConfig.ts` | 15-26 |
| at-capacity 0-or-≥100 | `src/bridge/pollConfig.ts` | 21-25 |
| capacityWake | `src/bridge/capacityWake.ts` | 全 56 行 |
| RemoteSessionManager | `src/remote/RemoteSessionManager.ts` | 全 343 行 |
| SessionsWebSocket 协议 | `src/remote/SessionsWebSocket.ts` | 80-85 |
| WebSocket 重连参数 | `src/remote/SessionsWebSocket.ts` | 18-36 |
| sdkMessageAdapter | `src/remote/sdkMessageAdapter.ts` | 全 302 行 |
| remotePermissionBridge | `src/remote/remotePermissionBridge.ts` | 全 78 行 |
| createDirectConnectSession | `src/server/createDirectConnectSession.ts` | 9-15 |
| DirectConnectSessionManager | `src/server/directConnectManager.ts` | 42-78 |
| 录音架构 | `src/services/voice.ts` | 6-12 |
| audio-capture-napi 阻塞 | `src/services/voice.ts` | 14-20 |
| voiceStreamSTT 协议 | `src/services/voiceStreamSTT.ts` | 1-12 |
| GLOBAL_KEYTERMS | `src/services/voiceKeyterms.ts` | 14-26 |
| voice mode gates | `src/voice/voiceModeEnabled.ts` | 8-15 |
| `/voice` 命令 | `src/commands/voice/voice.ts` | 全 ~50 行 |
| useSSHSession | `src/hooks/useSSHSession.ts` | 1-9 |
| useTeleportResume | `src/hooks/useTeleportResume.tsx` | 14-25 |
| `/teleport` stub | `src/commands/teleport/index.js` | 1 |
| `/mobile` 二维码 | `src/commands/mobile/mobile.tsx` | 13-22 |
| `/desktop` Handoff | `src/commands/desktop/desktop.tsx` | 全 |
| upstreamproxy 容器装配 | `src/upstreamproxy/upstreamproxy.ts` | 1-22 |
| CONNECT-over-WS relay | `src/upstreamproxy/relay.ts` | 1-18 |

---

## 本章小结

第十九章把 Claude Code 从"终端工具"扩展到"跨设备 Agent 平台"。这一章的核心要点：

**1. Bridge 是 Web 模式的本机代驾**——不是远程执行，而是让 Web UI 和本机的 Agent 进程对话。30 个文件、12,613 行处理认证、消息搬运、轮询节奏、批量边界、容量管理。

**2. JWT + trustedDevice 是双重信任锚**——JWT 校验消息合法、trustedDevice 校验设备授权。两者缺一不可，两段独立的 rollout flag 保证生产升级安全。

**3. inboundMessages / inboundAttachments 是兼容性翻译层**——把 Web/Mobile 客户端的不一致格式（camelCase vs snake_case）归一化进 Agent 内核。80 行 + 175 行的"小文件"卡住了一整类跨语言协议 bug。

**4. FlushGate 用 4 个生命周期方法解决 flush/重连边界的消息一致性**——deactivate vs drop 的语义区分把"丢/不丢"的决策外显出来，强制调用方明确选择。

**5. pollConfig 用 Zod schema + cross-field refine 把运维配置错误挡住**——0-or-≥100 / fat-finger 防御 / 一坏全坏的回退策略，每一条都是事故复盘后的产物。

**6. capacityWake 用 AbortSignal 实现可重置的 wakeup**——56 行小文件消除 replBridge / bridgeMain 两边 byte-for-byte 重复的逻辑。

**7. Remote / Server / Direct Connect 是三种"传输适配"**——Remote 接 Anthropic Server 长连接、Server 接用户自己跑的 server、SSH 接子进程。三者共享同一个 SDKMessage 流抽象，所以 Agent 内核不需要适配。

**8. Voice 1,229 行是平台级工程**——native module dlopen 阻塞、多平台音频后端、流式 STT 协议、领域 keyterms 提示、build-time + runtime 双 gate。

**9. SSH 模式让"UI 在本机、Agent 在远程"透明化**——auth proxy 通过 reverse port forward 让远程进程用本机 token，不需要 token 离开本机。

**10. Teleport / Dispatch 是两种会话延续模式**——Teleport 把当前 session 搬到另一台设备继续；Dispatch 派出独立的远程 agent 做后台任务。

**11. UpstreamProxy 是企业级安全的微缩**——MITM CA 注入、token 内存隔离、protobuf-over-WebSocket 路由穿透。普通用户不接触，但反映了 Claude Code 作为企业产品的成熟度。

**12. 跨设备的"无缝感"是被三大保证撑起来的**——稳定的全局 session ID、trustedDevice 信任链、frontend 完全可插拔。

读完这一章，你应该能理解：**Claude Code 不只是一个 CLI——它是一个可以跑在终端、Web、Mobile、SSH、企业沙箱里的同构 Agent**。终端只是它最早的 frontend，不是它唯一的 frontend。

### 本章用过的工程技巧速查

回顾一下这一章见到的"通用工程技巧"——这些不只适用于 Claude Code，做任何分布式系统都用得上：

| 技巧 | 出现位置 | 效果 |
|---|---|---|
| 依赖注入回避循环依赖 | bridgeApi.ts deps | 避免把 1300 个模块的冷启动开销带进 Bridge |
| 双 flag 渐进发布 | trustedDevice.ts | CLI 侧先发头，Server 侧后强制——降低发布风险 |
| 入口处归一化 | inboundMessages.ts | 单点容忍 camelCase / snake_case 不一致 |
| 状态机的双关闭语义 | flushGate.ts | drop（永久关）vs deactivate（暂关，pending 保留）|
| schema 一坏全坏 | pollConfig.ts | 比 clamp 更安全的回退策略 |
| 跨字段约束 | pollConfig.ts | refine 强制"至少一个 liveness 机制启用" |
| AbortSignal 合流 | capacityWake.ts | 多路信号合并到一路 abort |
| 可重置 controller | capacityWake.ts | wake() = abort 旧的 + new 一个 |
| double-check pattern | capacityWake.ts | 预先检查 aborted 防 race |
| 协议第一帧 auth | SessionsWebSocket.ts | 比 query string / header 更隐私 |
| 短暂错误码特殊重试 | SessionsWebSocket.ts | 4001 retry 3 次穿越 compaction 窗口 |
| forward-compatible type guard | SessionsWebSocket.ts | 不死板 allowlist，让新类型透传 |
| NDJSON streaming | directConnectManager.ts | partial-parse 友好的 stream 协议 |
| native module lazy load | voice.ts | 延迟昂贵的 dlopen 到首次使用 |
| 实际探测 vs PATH 检查 | voice.ts (probeArecord) | 跨平台软件的"假阳性"防御 |
| 双 gate（build + runtime）| voiceModeEnabled.ts | feature flag 两层控制 |
| 接口同形不同实现 | useSSHSession + useDirectConnect | 让 REPL 不感知 transport |
| auth proxy via reverse port forward | claude ssh | 远程进程用本机 token，不暴露 |
| git bundle + patch transport | gitBundle.ts | uncommitted 改动跨设备搬运 |
| MITM CA 注入 + token unlink | upstreamproxy.ts | 容器级安全的多步骤装配 |
| 多种 NO_PROXY 写法 | upstreamproxy.ts | 适配不同 runtime 的解析语义 |

这 21 条里大多数都是被生产事故"喂"出来的——下次你看到一个看似简单的代码却带着复杂注释，多读两遍，那里大概率藏着一个被血泪验证过的工程教训。

### 一些值得自问的问题

读完本章，你可以拿这些问题自测：

1. **为什么 Bridge 用 HTTP 长轮询而不是 WebSocket？** 提示：穿透代理 / 防火墙、运维简单度。
2. **为什么 Mobile 反而用 WebSocket？** 提示：低延迟、省电、native socket 支持。
3. **为什么 trustedDevice 信任的是"设备"而不是"浏览器"？** 提示：让你能在多个浏览器间切换不掉信任。
4. **为什么 capacityWake 用 AbortSignal 而不是 setTimeout/clearTimeout？** 提示：可组合、生命周期清晰、和 Web Standard async API 对齐。
5. **为什么 voice 的 native module 要延迟加载？** 提示：dlopen 同步阻塞，1-8 秒，不能让所有用户为可能不用的功能等。
6. **为什么 SSH 模式要起 auth proxy 而不是把 token copy 到远程？** 提示：避免 token 离开本机，安全模型清晰。
7. **为什么 Teleport 用 git bundle + patch 而不是 tar workspace？** 提示：尺寸效率、gitignore 原生支持、增量上传可能。
8. **upstreamproxy 为什么"fail open"？** 提示：proxy 是增强不是必需，崩了不该让 session 整个死。

如果这 8 个问题你能流畅回答 6 个以上，说明这一章读得相当扎实。

### 写在这一章末尾

Claude Code 这套"可以在任何地方跑的 Agent"听起来像营销语，但读完源码会发现它**真的成立**——不是靠 marketing，是靠把 transport 抽象、把 frontend 抽象、把信任链外置出来。

Anthropic 的工程师在每一个边界（client/server、本机/远端、可信/不可信）都做过认真的取舍。你不需要赞同每一个具体决策，但理解他们为什么这么决策，对你设计自己的分布式系统会有帮助。

### 2026 演进补充：Cloud Scheduled Tasks、Channels 与实时流式远控（v2.1.198–v2.1.326）

在 `v2.1.88` 打下的 Bridge 与 Remote 同构底座之上，2026 年下半年跨设备与云端自治产品线完成了三块关键拼图：

1. **Routines 正式升级并更名为 Cloud Scheduled Tasks（2026 年 10 月）**：面向无人值守场景的云端托管沙箱代理（保留 `/routines`、`/schedule` CLI 命令与 `POST .../fire` Webhook 端点），支持 **Cron 定时触发**（最小间隔 1 小时）、**API Bearer Token Webhook 触发** 与 **GitHub 事件触发**（如 `pull_request.opened`、Issue、Release），每次运行自动全新克隆仓库并在 Auto Mode 下独立提交 PR。
2. **Channels 双向事件推流成熟化**：与“在云端另起隔离容器”的 Cloud Scheduled Tasks 互补，`claude --channels plugin:telegram@claude-plugins-official`（覆盖 Telegram、Discord、iMessage、Slack 及 CI/Sentry 告警）通过 MCP Server 将外部消息或生产告警**直接推入本机正在运行的会话**，无缝复用本机未提交代码与本地环境。
3. **Remote Control 实时 Token 流与多端扩展（`v2.1.198`–`v2.1.326`）**：`claude remote-control` 新增 `--settings <file>` 配置注入（`v2.1.321`），并在 `v2.1.326` 实现了向手机端与 Web 浏览器的**逐 Token 实时流式回复推送**；与此同时，**Claude in Chrome** 浏览器代理正式 GA（`v2.1.198`），**Claude Desktop Linux 版**开放公测，共享 Artifacts 实时预览也在 `v2.1.178+` 打通。

到这里，前六个模块的源码拆解已经全部收官。从下一章开始，我们将进入**模块七：实战项目**——不再只是读别人的代码，而是亲手从零写出属于你自己的 `MiniAgent`（Ch20）、自定义 `MCP Server`（Ch21）与多代理协作系统（Ch22）。

---

## 思考题

Bridge 12,613 行代码主要解决的问题中，哪一个对你的项目最有借鉴价值？

欢迎在评论区聊聊你的想法。

下一讲，我们换个角度看《项目一：MiniAgent》。

