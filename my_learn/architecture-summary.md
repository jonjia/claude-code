# Claude Code 架构总览

---

## 一句话心智模型

**Claude Code = Agent Loop + Tools + Terminal UI**，三者通过分层调度协作。用户输入一条消息后，进入一个 turn loop：模型说话 → 调工具 → 把工具结果塞回去 → 再次问模型 → 直到模型说"我说完了"。

---

## 完整架构分层

```
┌─ cli.tsx (分发器，feature-gated 快速路径) ─────────────────┐
│   --version / daemon / RCS / computer-use-mcp / ...        │
│   └─ 默认路径: await import('./main.tsx')  [按需加载]      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─ main.tsx (Commander.js CLI) ─────────────────────────────┐
│   注册 subcommands、启动 REPL                              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─ REPL.tsx (Ink/React) ────────────────────────────────────┐
│  <Static>历史消息</Static>                                  │
│  <流式 assistant / PermissionCard / PromptInput>           │
│                                                             │
│  state 通道:                                               │
│  ├─ AppState (Zustand)：messages, pendingPermission, ...   │
│  └─ bootstrap singleton：sessionId, CWD, token counts      │
└─────────────────────────────────────────────────────────────┘
                          ↓ 用户回车触发
┌─ QueryEngine ─────────────────────────────────────────────┐
│  session 级：历史、compaction、快照、/clear                 │
└─────────────────────────────────────────────────────────────┘
                          ↓ per user prompt
┌─ query.ts (async generator) ──────────────────────────────┐
│  turn loop：while(stop_reason === 'tool_use')              │
│   - await canUseTool(...) ← Promise/resolver 桥接 UI       │
│   - 并发执行工具                                            │
│   - yield 消息给上层 UI                                     │
└─────────────────────────────────────────────────────────────┘
                          ↓ per API call
┌─ services/api/claude.ts ──────────────────────────────────┐
│  拼 system prompt（稳定在前 + CLAUDE.md + gitStatus）      │
│  选 provider → 调 adapter                                  │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─ Provider Adapters ───────────────────────────────────────┐
│  firstParty / bedrock / vertex / foundry / openai / gemini │
│  / grok                                                    │
│  把各家 streaming 翻译成 Anthropic BetaMessage             │
└─────────────────────────────────────────────────────────────┘

横切关注点：feature('X') Bun 构建期替换 + DCE，禁用代码不进产物
```

---

## 10 个核心概念

### 1. 整体心智模型

三个必然存在的子系统协作：
- **A. LLM 对话管理**（发请求、接流、记历史）
- **B. 工具执行**（模型说要读文件，谁去读？谁决定能不能读？）
- **C. 用户界面**（输入/显示/权限弹窗）

它们必须由**一个中心调度者**串起来，因为彼此异步等待（流式 token、工具执行、用户审批）。

---

### 2. 入口与启动流程

**`src/entrypoints/cli.tsx`** 是真入口，它是一个**带快速路径的分发器**：

```
1. --version / -v        ← 零模块加载
2. --dump-system-prompt
3. --computer-use-mcp    ← 独立 MCP server 模式
4. remote-control / rc   ← Bridge 模式
5. daemon [sub]
6. ps / logs / attach
...
N. 默认路径 → 动态 import('./main.tsx')
```

**关键洞察**：cli.tsx 的每条快速路径 = 一种独立运行模式。**main.tsx 只是"交互式 Claude Code"这一个模式的实现**。这种分层让 `claude --version` 0 模块加载、`claude remote-control` 只加载 bridge 代码。

---

### 3. Turn Loop 核心（query.ts）

```ts
export async function* query(messages, tools, ...): AsyncGenerator<...>
```

注意是 **async generator**（带 `*`）。原因：调用者不能干等模型跑完 10 轮工具，generator 让 query.ts 每出一小段内容就 `yield` 给消费者，实现"一字一字打出来"的流式渲染。

**循环终止条件**：模型在流式响应结束时返回 `stop_reason`：
- `"tool_use"` → 执行工具，追加 tool_result，**继续循环**
- `"end_turn"` → **退出循环**
- `"max_tokens"` / `"stop_sequence"` → 其他终止

**工具参数是流式拼出来的**：
```
content_block_start    { type: "tool_use", name: "FileEditTool" }
content_block_delta    input_json_delta: "{\"path\": \"/tmp"
content_block_delta    input_json_delta: "/foo.ts\", ...}"
content_block_stop     ← 此时才 parse 完整 JSON 并触发工具执行
```

UI 在 delta 阶段实时渲染拼接结果，但**真正执行工具必须等 block_stop**。

模型可以在一次响应里**并行返回多个 tool_use blocks**，query.ts 会**并发执行**它们，把多个 tool_result 一起打包塞回下次请求。

---

### 4. QueryEngine：会话级编排

**`src/QueryEngine.ts`** 包裹 query()，管理跨多次 user prompt 的事：

| 职责 | 归属 |
|---|---|
| 流式拼 JSON、tool_use/tool_result turn 内装配 | query.ts |
| 历史保存、compaction、/clear、文件快照 | QueryEngine |
| 渲染、权限弹窗 | REPL |

**关键**：`query.ts 是 stateless 的**，只接 messages 数组、产出消息。

---

### 5. 工具系统 + 权限

**两层结构**：
- `src/Tool.ts` — Tool 接口（name、description、input schema、call、权限规则）
- `src/tools.ts` — 注册表（按 feature flag / env 条件加载）
- `packages/builtin-tools/` — 60 个具体工具（FileEdit、Bash、Grep、TaskCreate、AgentTool…）

**`canUseTool` 回调** = 权限策略模式。REPL 把一个 `canUseTool(name, input) → Promise<Result>` 函数传给 query()，query.ts 在执行每个工具前 `await` 它。

**同一个 query.ts 适配多种模式**：

| 模式 | canUseTool 实现 |
|---|---|
| 交互 REPL | 弹 React 权限卡片，等用户点击 |
| `claude -p` pipe | 按 allowlist 判断，自动 resolve |
| `--dangerously-skip-permissions` | `() => ({ allow: true })` |
| Remote Control / Bridge | 通过 WebSocket 发到远端，回传后 resolve |
| 测试 | mock 成可预测响应 |

**query.ts 一行不改**。这是策略模式 + 依赖倒置。

---

### 6. UI 层：Ink + React

**为什么用 React 写 CLI？** Claude Code 的界面本质是**有状态 UI**：流式消息、可编辑输入框、权限弹窗、spinner、状态栏。手动管理终端 ANSI 控制序列是噩梦。

**Ink = React renderer for terminal**。`<Box>`、`<Text>` 组件，diff 后翻译成 ANSI。

**`<Static>` 边界 — 性能命脉**：

```jsx
<Static>
  {历史消息 1}        ← 渲染一次后固化，不再 diff
  {历史消息 2}
  ...
</Static>

<Box>{流式 assistant}</Box>   ← dynamic
<PermissionCard />            ← dynamic（出现/消失）
<PromptInput />               ← dynamic（光标响应）
```

否则 200 轮对话后每个新 token 都要 diff 整棵树，CPU 爆炸。

**Promise ↔ React state 桥接**（权限弹窗机制）：

```
query.ts:                     React UI:
─────────                     ──────────
await canUseTool()
  └─ new Promise((resolve) => {
       appState.pendingPermission = {
         req, resolve              ← resolver 暂存到 state
       }
     })
                               re-render → <PermissionCard />
                                              │
                                        用户点"允许"
                                              │
                                      onClick → resolver({allow:true})
  ◄────────────────────  清空 state，卡片消失
继续执行工具
```

**并发权限的处理**：query.ts 可以并发 await 多个 canUseTool，UI 侧**串行排队弹卡片** — 终端高度有限、键盘焦点管理简单。query.ts 只关心"两个 Promise 都 resolve 后才执行"，不关心 UI 是串行还是并发。

---

### 7. 状态管理：双容器

| 容器 | 内容 | 实现 |
|---|---|---|
| **AppState** | messages, pendingPermission, MCP 连接, 模态开关 | Zustand-style store + React Context（响应式） |
| **bootstrap/state.ts** | sessionId, CWD, project root, token counts, model overrides, permission mode | 模块级变量 + getter/setter（不响应式） |

**反直觉点**：token counts 在 bootstrap 而不是 AppState。因为每个流式 event 都累加，频率极高，放响应式 store 会**每个 token 都触发整个状态栏 re-render**。bootstrap 里纯赋值零成本，UI 用一个节流的 hook（~100ms）读取显示。

> **原则让位于性能**：要显示的状态一般放响应式，但更新频率高到一定程度，必须拆分快速通道 + 慢速订阅。

---

### 8. 上下文构建

**`src/context.ts`** 自动注入模型需要的上下文：
- `getGitContext()` → `{ gitStatus }`
- `getClaudeMdContext()` → `{ claudeMd }`
- `getDateContext()` → `{ currentDate }`
- `getDirectoryStructure()` → 项目浅层目录树

**CLAUDE.md 分层** — `src/utils/claudemd.ts` 沿目录层级搜索：
```
~/.claude/CLAUDE.md          ← 用户全局规则
└── /project/CLAUDE.md       ← 项目规则
    └── /project/src/CLAUDE.md  ← 子目录规则
```
全部拼接作为系统规则。

**字段刷新频率**：

| 字段 | 频率 | 原因 |
|---|---|---|
| CLAUDE.md | 启动时缓存，/memory 编辑后重载 | 文件不变就不重读 |
| currentDate | 每次 turn 计算 | 代价为 0 |
| gitStatus | 每次 turn 前重跑 git | 工具可能刚改了文件 |

**Prompt Caching 决定字段顺序**（前缀相同的 prompt 命中缓存，成本降 90%）：

```
[system prompt 静态部分：工具描述、通用规则]   ← 缓存几乎永久命中
[CLAUDE.md 合并内容]                          ← session 内稳定
[directoryStructure]                          ← session 内较稳定
---- cache_control 边界 ----
[currentDate、gitStatus]                      ← 每 turn 变
[messages 数组]                               ← 不断增长
```

> **稳定在前，变化在后** — 这是真金白银的优化。

---

### 9. Provider 抽象：Stream Adapter Pattern

支持 7 个 provider：firstParty、bedrock、vertex、foundry、openai、gemini、grok。

**核心挑战**：OpenAI、Gemini、Grok 的 API 与 Anthropic 不同。但 query.ts 只想面对**一种**数据形状。

**解法**：每个 provider 一个 adapter，把下游 streaming events 实时翻译成 Anthropic BetaRawMessageStreamEvent。

```
query.ts  （只吃 Anthropic BetaMessage 格式）
    │
services/api/claude.ts  （调用当前 provider adapter）
    │
 ┌──┴──────────┬─────────┬─────────┬──────┐
 ▼      ▼       ▼         ▼         ▼      ▼
fp   bedrock  openai   gemini    grok   ...
                (翻译 tool_calls↔tool_use, delta 格式, system 字段, auth)
```

**adapter 吸收的差异**：
- tool use 协议（`tool_calls` 数组 vs `content` block 里的 `tool_use`）
- streaming event 名称和结构
- system prompt 位置（独立字段 vs 第一条 message）
- 认证 header / endpoint

Provider 选择优先级：`modelType` 参数 > 环境变量 > 默认 firstParty（见 `src/utils/model/providers.ts`）。

---

### 10. Feature Flag 系统

**`bun:bundle` 是 Bun 构建器的虚拟模块**，不是真模块。

```ts
import { feature } from 'bun:bundle'

if (feature('BRIDGE_MODE')) {
  await startBridge()
}
```

**机制 = 编译期常量替换 + dead code elimination**：

```ts
// BRIDGE_MODE 启用 → 替换为：
if (true) { await startBridge() }
// → 简化为：await startBridge()

// BRIDGE_MODE 禁用 → 替换为：
if (false) { await startBridge() }
// → DCE → 代码从产物里完全消失
```

**语法限制**（CLAUDE.md 强调）：
- `feature()` **只能**直接放在 `if` 或三元表达式条件位置
- 不能 `const x = feature('X')`
- 不能放在箭头函数体里
- 不能作为 `&&` 链的一部分

**为什么有这些限制？** 因为它是编译期替换，不是真函数 — 必须能被静态识别才能替换。

**典型用法（动态 import）**：

```ts
if (feature('DAEMON')) {
  const { startDaemon } = await import('./daemon/main.ts')
  startDaemon()
}
```

DAEMON 关闭时，整个 if 被消除 → `import('./daemon/main.ts')` 从未执行 → daemon 代码不进产物。

**默认启用列表**：
- **Build 模式**：19 个（见 `build.ts`）— 生产产物只带这些
- **Dev 模式**：全部启用（见 `scripts/dev.ts`）— 开发时全部代码可用
- **运行时启用**：`FEATURE_<FLAG_NAME>=1` — 但只在 dev/build 阶段由构建器读取，禁用 feature 的代码运行时无法再开启（已经不在产物里）

---

## 三个反复出现的架构主题

### 1. 核心稳定，变化关进盒子
- **canUseTool**：query.ts 不变，换 callback 实现适配交互/headless/远程
- **Stream adapter**：query.ts 只吃一种格式，adapter 翻译各家差异
- **Feature flag**：核心循环不知道有多少模式，构建期决定哪些代码进产物

### 2. 分层依赖签名而非实现
```
REPL → QueryEngine → query.ts → claude.ts → SDK
```
每层只依赖下层的接口签名，不依赖具体实现。query.ts 不知道 REPL 是谁，QueryEngine 不知道 provider 是谁。

### 3. 原则让位于性能
- token counts 放 bootstrap 而不是响应式 AppState
- `<Static>` 边界把已完成的对话固化，避免无谓 diff
- prompt cache 顺序：稳定在前、变化在后

---

## 三个常见误区

### 误区 1：独立运行模式应作为 main.tsx 子命令
**实际**：cli.tsx 每条快速路径 = 一种独立运行模式。daemon、RCS、MCP server 这些**不需要 REPL**，加载 main.tsx 是浪费。

### 误区 2：处理 pipe 模式权限应加 `skipPermission` flag
**实际**：换一个 `canUseTool` 实现即可，query.ts 一行不改。否则面对 "pipe 模式 + allowlist" "远程模式 + WebSocket" 等组合会无限加 flag。

### 误区 3：`feature()` 是运行时开关
**实际**：是构建期常量替换。禁用的 feature 的代码**根本不在产物里**，运行时无法开启。这也是 cli.tsx 快速路径能"零模块加载"的关键。

---

## 关键代码索引

| 文件 | 职责 | 行数 |
|---|---|---|
| `src/entrypoints/cli.tsx` | 入口分发器、快速路径 | ~373 |
| `src/main.tsx` | Commander CLI、subcommands、REPL 启动 | ~6981 |
| `src/screens/REPL.tsx` | 交互式 UI、权限审批 | - |
| `src/QueryEngine.ts` | 会话级编排 | - |
| `src/query.ts` | turn loop、async generator | ~1773 |
| `src/services/api/claude.ts` | API 客户端、streaming events | ~3485 |
| `src/Tool.ts` | Tool 接口定义 | - |
| `src/tools.ts` | 工具注册表 | ~392 |
| `packages/builtin-tools/` | 60 个工具实现 | - |
| `src/state/AppState.tsx` | 集中式响应式状态 | - |
| `src/bootstrap/state.ts` | 模块级单例 | - |
| `src/context.ts` | 上下文构建（git、CLAUDE.md、date） | ~189 |
| `src/utils/claudemd.ts` | CLAUDE.md 分层加载 | ~1480 |
| `src/utils/model/providers.ts` | provider 选择 | - |
| `packages/@ant/ink/` | Fork 的 Ink 框架 | - |
| `build.ts` | 构建脚本（feature flag 默认列表） | - |
| `scripts/dev.ts` | dev mode（全 feature 启用） | - |
| `scripts/defines.ts` | MACRO 常量集中管理 | - |

---

## 进一步学习方向

- **Agent 子 agent 系统**（AgentTool、Task、subagent 调度）
- **Bridge / RCS 远程控制架构**（WebSocket、session relay、权限穿透）
- **MCP 客户端实现**（工具动态注入、OAuth）
- **Prompt caching 的精确锚点机制**（cache_control 边界放置策略）
- **端到端代码走读**（"读 foo.ts" 从输入到屏幕出字的完整 trace）
- **Compaction 策略**（token 超限时如何用模型总结历史）

---

*整理自 2 次 /teach-me 互动学习 session（2026-04-20 → 2026-04-21），10 个核心概念全部掌握。*
