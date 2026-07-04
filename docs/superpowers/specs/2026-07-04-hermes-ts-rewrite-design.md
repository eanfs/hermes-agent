# Hermes-TS 重写设计（Spec）

- 日期：2026-07-04
- 状态：待评审
- 作者：lirichen + Claude
- 决策方式：brainstorming 流程逐节确认

## 1. 背景与动机

hermes-agent（Python 主体约 132 万行 + TS/TSX 约 26 万行）是一个多渠道、模型无关的
个人 AI agent。本次重写有两个动机：

1. **换技术栈**：从 Python + Node/React 混合栈迁移到 TypeScript 全栈（Node ≥ 22，ESM）。
2. **学习目的**：通过重写深入理解 agent 系统的架构设计。

产出质量以"架构清晰、可运行、可演进"为准，不追求短期内替代 Python 版的全部功能。

## 2. 现状评估（重写要修的结构病）

| 病灶 | 现状证据 |
|---|---|
| 上帝文件 | `gateway/run.py` 20k 行、`cli.py` 16k、`hermes_cli/web_server.py` 15k、`hermes_cli/main.py` 14k、`tui_gateway/server.py` 14k |
| 构造器失控 | `AIAgent.__init__` 约 60 个参数，凭证/路由/回调/预算/会话混在一处 |
| 同步/异步失配 | 同步的 `run_conversation` 主循环被异步 gateway 与平台适配器驱动 |
| 双重平台层 | `gateway/platforms/` 与 `plugins/platforms/` 两套适配层，边界不清 |
| import 副作用 | 工具靠 import 时自注册，加载顺序是隐性依赖 |
| 配置蔓延 | 示例配置 72KB + env 示例 23KB，多处合并逻辑 |

旧代码同时也是最完整的"需求文档"：`AGENTS.md`（71KB）为权威架构参考，
各平台适配器沉淀了大量平台脾气（如 Telegram editMessageText 限速）。

## 3. 目标与非目标

### v1 范围（已确认）

- Agent 核心循环 + 工具系统 + 多 Provider（OpenAI 兼容、Anthropic）
- SQLite 会话存储 + FTS5 全文搜索
- 分层配置系统
- Gateway（消息路由、会话管理）
- Telegram 平台适配器
- CLI 交互入口
- 旧版 config / 会话数据的一次性迁移导入工具
- skills 原生采用 agentskills.io 开放标准的**格式**（v1 仅保证格式兼容的读取不报错，执行引擎属非目标）

### 非目标（v1 明确不做）

- 其余 19 个聊天平台、cron、记忆系统、kanban、ACP、桌面端、Web 面板、
  语音、训练数据工具链（batch runner / trajectory compressor）
- 六种终端后端（v1 仅本地 shell 工具）
- 运行时直接读取旧版配置/数据库文件（兼容通过一次性导入实现，不背旧 schema）

## 4. 路线选择（已确认：方案 A）

**绿地重写 + 概念移植**：新建独立 TS monorepo，以旧项目源码与 `AGENTS.md`
作为需求文档，移植行为与概念而非逐行翻译。借用 Strangler Fig 的纪律：
按垂直切片推进里程碑（每个里程碑端到端可运行），关键行为写对照测试。

否决的备选：
- **B（Strangler Fig 双语并行）**：跨语言 IPC 的管道工程稀释学习收益，适合团队生产系统而非个人项目。
- **C（逐模块忠实翻译）**：会原样继承上帝文件与 60 参数构造器，与两个动机均背道而驰。

## 5. 总体架构

**风格：六边形架构（Ports & Adapters）+ 事件驱动核心。**
kernel 只定义端口接口，一切外部世界（LLM、平台、存储）都是可替换适配器。

```
hermes-ts/  (pnpm workspaces + TypeScript strict + ESM + Vitest)
├── packages/
│   ├── kernel/              # 核心域：Agent 循环、消息模型、事件类型
│   │   └── 定义端口: ProviderPort, ToolPort, StoragePort, ChannelPort
│   ├── providers/           # openai-compatible, anthropic（实现 ProviderPort）
│   ├── tools/               # 工具注册表 + 内置工具（显式注册）
│   ├── storage/             # SQLite + FTS5（实现 StoragePort）
│   ├── config/              # 分层配置 + zod schema
│   ├── gateway/             # 消息路由、会话管理（平台无关）
│   └── platform-telegram/   # grammY 适配器（实现 ChannelPort）
├── apps/
│   └── cli/                 # CLI 入口（直连 kernel，不经 gateway）
└── migration/               # 一次性导入：旧 config.yaml / 旧 SQLite → 新格式
```

新代码库为独立仓库（不放在本 Python 仓库内）；本 spec 与实施计划保存在
当前 fork 中作为规划记录。

### 四个核心决策 → 对应病灶

1. **异步优先**：kernel 原生 async，流式响应、工具并发、多会话是一等形态 → 修同步/异步失配。
2. **配置对象 + 依赖注入**：`new Agent(config, ports)`，参数按内聚性分组，外部依赖显式注入 → 修 60 参数构造器；CLI 与 gateway 用不同注入组合复用同一 kernel。
3. **单一平台适配层**：只有 `ChannelPort` 一个接口，Telegram/未来的 Discord/CLI 均为其实现 → 修双重平台层。
4. **事件驱动对外**：kernel 产出 `AgentEvent` 流，渲染/转发都是下游消费者 → 上帝文件失去土壤。

## 6. Kernel 设计

### 消息模型

内部中立格式 `ModelMessage` / `ContentPart`（text、tool-call、tool-result、image），
不以 OpenAI wire format 作为内部货币；Provider 适配器负责双向翻译。

### Agent 循环

单个异步生成器（目标量级 ~200 行）：

```
async *runTurn(input): AsyncGenerator<AgentEvent>
  循环（直到无工具调用 或 迭代预算耗尽）:
    1. provider.stream(messages) → 转发 text-delta / tool-call 事件
    2. 收集工具调用 → 并发执行（AbortSignal 支持中断）
    3. 工具结果追加进 messages → 下一轮
  预算耗尽时发起一次收尾调用（继承旧版 grace call）
```

事件类型（初版）：`turn-start`、`text-delta`、`tool-call-start`、`tool-call-end`、
`turn-complete`、`error`、`budget-exhausted`。中断/超时/预算统一走
`AbortSignal` + 计数器，不用轮询标志位。

### 工具系统

`Tool = { name, description, parameters: ZodSchema, execute(args, ctx) }`。

- zod 一份定义同时产出 JSON Schema（给 LLM）与运行时校验（给执行层），消除双份维护漂移。
- `createToolRegistry([...tools])` 显式组装，无 import 副作用。
- `ToolContext` 注入：会话引用、工作目录、事件发射器、权限检查钩子。
- v1 内置工具（3 个）：shell 执行、文件读、文件写。网页抓取等进 v2 backlog。

### Provider 抽象

`ProviderPort.stream(request): AsyncIterable<ProviderEvent>` —— 一个方法的接口。
流式 tool-call 增量拼装、重试退避、prompt caching 标记收在适配器内部。
v1 实现：OpenAI 兼容（覆盖 OpenRouter 等）+ Anthropic 原生。

## 7. Gateway 与数据流

### 消息旅程

```
Telegram update
  → platform-telegram 归一化为 InboundMessage { channel, conversationId, sender, parts[] }
  → gateway Router: (channel, conversationId) → 查找/创建 Session
  → SessionManager: 载入历史，调用 kernel.runTurn()
  → AgentEvent 流 → ChannelRenderer（格式转换、分片、流式编辑节流、媒体）
  → adapter.send() 回 Telegram
```

### 关键设计

1. **Adapter 与 Renderer 分离**：adapter 只管传输与归一化；renderer 是纯函数层
   `AgentEvent[] → PlatformMessage[]`，可无 IO 单测。
2. **会话并发**：每会话同时只有一个 agent run；SessionManager 内显式状态机
   `idle → running → interrupting`；运行中新消息排队，中断指令映射 AbortSignal。
3. **CLI 不经 gateway**：直连 kernel 消费同一事件流，保证 kernel 对 gateway 零感知。
4. **领域知识挖掘**：实现 Telegram 适配器前，先从旧版 8k 行 adapter 中提炼
   平台脾气清单（限速、分片阈值、Markdown 方言等）为设计笔记。

## 8. 存储、配置与错误处理

### 存储（better-sqlite3，薄 DAO，无重 ORM）

- 表：`sessions`（元数据 + 平台绑定键）、`messages`（追加写入，存 ModelMessage JSON）、
  FTS5 虚拟表（全文搜索）。WAL 模式。
- **版本化迁移器**：`migrations/00N.sql` 启动按序执行（替代旧版运行时 schema 修补）。
- `migration/` 导入工具：旧 SQLite → 新 schema，一次性，历史对话不丢。

### 配置（zod 单一真相源，启动快速失败）

- 分层：内置默认 → `config.yaml` → 环境变量 → CLI 参数；秘钥只放 env。
- 一个 zod schema 同源产出校验、类型、文档；启动时报人类可读错误。
- 纪律：**每个配置项必须有消费者**；v1 目标 ≤ 30 个键。
- `migration/` 提供旧 `~/.hermes/config.yaml` 的映射导入。

### 错误处理（按"谁能恢复"分类，禁止静默失败）

| 类别 | 策略 |
|---|---|
| Provider 错误（限流/超时） | 适配器内指数退避重试，超限后作为 error 事件浮出 |
| 工具执行错误 | **是数据不是异常**：包装成 tool-result 返给 LLM，由 agent 自行恢复 |
| 会话级致命错误 | 杀会话不杀进程，错误事件投递回用户所在平台 |
| 平台投递错误 | 有限重试 + 结构化日志，不进空 catch |

日志：pino 结构化输出，事件携带 sessionId 关联。

## 9. 测试策略

1. **Kernel 单测 + FakeProvider**（核心投资）：脚本化吐事件，覆盖多轮工具调用、
   并发、中断、预算收尾，零网络毫秒级。
2. **端口契约测试**：ProviderPort / ChannelPort 各一套共享套件，新适配器必须通过。
3. **Renderer 纯函数测试**：格式转换、分片、节流全覆盖。
4. **对照测试（parity）**：从 Python 版录制关键行为样本（工具循环轨迹、
   搜索结果）作黄金用例。
5. **集成冒烟**：内存 SQLite 全链路 + CLI e2e。
6. 全程 TDD（superpowers 流程）。

## 10. 里程碑（垂直切片，每步端到端可用）

| 里程碑 | 交付物 | 完成标志 |
|---|---|---|
| M0 脚手架 | pnpm monorepo、TS strict、Vitest、CI、lint | `pnpm test` 全绿 |
| M1 会说话的核心 | kernel + OpenAI provider + 3 工具 + 内存存储 + 最小 CLI | 终端完成带工具调用的对话 |
| M2 有记忆的核心 | SQLite + FTS5 + 配置系统 + Anthropic provider | 重启恢复会话、可搜历史 |
| M3 上 Telegram | gateway + ChannelPort + Telegram 适配器 | 手机上与 agent 对话 |
| M4 迁移与收尾 | config/会话导入、对照测试、文档 | 旧数据一键迁入，parity 全绿 |

**每个里程碑内的循环**：挖旧代码领域知识（产出一页设计笔记）→ TDD 实现 →
契约/对照测试把关 → 里程碑演示。架构含量最高的决策点（M1 事件类型设计、
M3 会话状态机等）由用户亲手实现，以达成学习目的。

## 11. 风险与对策

| 风险 | 对策 |
|---|---|
| 低估旧代码隐藏的边界情况 | 每里程碑先挖矿出设计笔记；对照测试兜底 |
| 绿地重写失去可运行状态 | 垂直切片纪律：任何时刻主分支端到端可用 |
| 范围蔓延（想顺手加 cron/记忆） | 非目标清单白纸黑字；新需求进 backlog 等 v2 |
| 配置面复发失控 | 每配置项必须有消费者 + zod 单 schema |

## 12. 后续步骤

1. 用户评审本 spec。
2. 通过后进入 superpowers:writing-plans，产出 M0+M1 的详细实施计划。
3. 新建 hermes-ts 独立仓库开工。
