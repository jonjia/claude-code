# Claude Code 功能全景图

> 一份自包含的 Claude Code 主要功能盘点，可独立阅读、可分享给团队。
> 适用对象：用过 REPL、想系统了解 Claude Code 全部能力的开发者。
> 一句话定位：**Claude Code 不只是一个 CLI 聊天框，而是一套覆盖交互/工具/扩展/记忆/多模型/远程的开发工作平台。**

---

## 0. 全景

```
┌──────────────────────────────────────────────────────────────────┐
│                      Claude Code 6 大能力面                       │
├──────────────────┬───────────────────────────────────────────────┤
│ 1. 交互模式       │ REPL / Headless / Plan / Auto                 │
│ 2. 内置工具       │ File / Bash / Web / MCP / Cron / Agent / Task │
│ 3. 扩展点         │ MCP / Skills / Hooks / Slash / Subagents      │
│ 4. 上下文与记忆   │ CLAUDE.md / memory / session / compaction     │
│ 5. 多 Provider    │ 7 家 + /login + 1M context + /poor            │
│ 6. 高级运行模式   │ daemon / RCS / Computer Use / Voice / Chrome  │
└──────────────────┴───────────────────────────────────────────────┘
```

不同能力解决不同问题：
- **交互**回答"怎么和它说话"
- **工具**回答"它能做什么动作"
- **扩展**回答"我怎么把自己的东西塞进来"
- **记忆**回答"它记得多久、记什么"
- **Provider**回答"用谁的模型、多少钱"
- **高级模式**回答"还能在哪里跑、怎么远程驱动"

---

## 1. 交互模式

| 模式 | 触发 | 何时用 |
|---|---|---|
| **REPL** | `claude` | 日常对话、探索性任务 |
| **Headless / Pipe** | `claude -p "..."` 或 `echo "..." | claude -p` | CI、脚本管道、单次问答 |
| **Plan Mode** | `/plan` 或 EnterPlanMode | 高风险变更前先对齐方案，**不直接改代码** |
| **Auto Mode** | `claude auto-mode` + 模板任务 | 无人值守批量任务（CI 化的 REPL） |

**配套命令**（Auto Mode 体系）：`new` / `list` / `reply` —— 模板化任务的创建、列表、回复。

---

## 2. 内置工具能力

约 50+ 个内置工具，按用途分类：

| 类别 | 工具 |
|---|---|
| **文件** | Read, Write, Edit, Glob, Grep, NotebookEdit |
| **执行** | Bash, PowerShell, REPL, Monitor |
| **Agent / Task** | Agent (fork / subagent), TaskCreate / Update / List / Get / Output / Stop |
| **规划** | EnterPlanMode, ExitPlanMode, VerifyPlanExecution |
| **Web** | WebFetch, WebSearch |
| **MCP** | MCPTool, McpAuthTool, ListMcpResources, ReadMcpResource |
| **调度** | CronCreate, CronList, CronDelete |
| **Worktree** | EnterWorktree, ExitWorktree |
| **辅助** | LSPTool, ConfigTool, SkillTool, AskUserQuestion, Snip, CtxInspect, Sleep, PushNotification, SendUserFile |

### 易被忽视、但值得多用的两个工具

**Agent**：
- `fork`（不指定 subagent_type）= 把当前对话克隆出去，**保留上下文**，结果不污染主 context
- `subagent_type=xxx` = 调用项目里预定义的 subagent（独立提示词 + 工具集）
- 适合：研究类问题、不关心中间过程的实现任务

**CronCreate**：
- 给未来某时刻排一个 prompt，可持久化
- 适合：提醒、定期巡检、定时任务

---

## 3. 扩展点（6 件套）

| 机制 | 触发方 | 何时定义 | 用途 |
|---|---|---|---|
| **MCP** | 模型调用工具 | settings.json + 服务器进程 | 接外部 API / 服务 |
| **Skills** | 模型按需检索 | `.claude/skills/<name>/SKILL.md` | 领域知识包 |
| **Hooks** | 生命周期事件 | settings.json + shell 脚本 | 拦截 / 增强工具调用、提交、消息提交等 |
| **Slash 命令** | 用户输入 `/foo` | `.claude/commands/<name>.md` | 用户级快捷 prompt |
| **Subagents** | Agent 工具调用 | `.claude/agents/<name>.md` | 隔离 context 的专用 agent |
| **Plugins** | 一次安装一组 | （本仓已简化掉） | 上述几个的捆绑分发 |

### 关键区别

- **谁触发它**：MCP / Skills / Hooks / Subagents = **模型或事件**触发；Slash = **用户**触发
- **进不进 system prompt**：Hooks 不进；Skills 元数据进、内容按需进；MCP 工具列表进
- **谁写**：MCP / Hooks 通常项目维护者写；Skills / Slash / Subagents 个人或团队都可

### 三组高频混淆

- **Hooks vs MCP**：Hooks 是 shell 脚本（同进程子进程），MCP 是常驻 server 进程
- **Skills vs Slash**：Skills 模型自己决定何时用（描述驱动），Slash 用户主动喊
- **Subagent vs fork**：fork 共享当前 context；subagent 独立 context + 独立提示词

---

## 4. 上下文与记忆

### 4 种持久化机制

| 机制 | 触发 | 存哪 | 何时用 |
|---|---|---|---|
| **CLAUDE.md** | 启动 + cwd 变化时自动加载 | 项目根 / 父目录 / `~/.claude/CLAUDE.md` | 项目级长期约定（团队共享） |
| **Auto memory** | 模型主动 Write + extract_memories 后台抽取 | `~/.claude/projects/<slug>/memory/*.md` | 跨会话个人偏好/项目动机/外部指针 |
| **Compaction** | context 接近上限自动触发 / `/compact` | 内存中重写 messages | 长会话防溢出 |
| **Session** | 每条消息追加 | `~/.claude/projects/<slug>/<uuid>.jsonl` | `--resume` / `--continue` 恢复 |
| **Snip / CtxInspect** | 模型按需调用 | 仅当前 context | 主动剪 tool output、查 token 占用 |

### 一定要记住的两条原则

**① CLAUDE.md = 注入 prefix（cache 友好）；memory = 按需读尾部**

→ 这就是为什么 CLAUDE.md 写多了不慢（命中 cache），memory 写多了也不慢（不读就不加载）。

**② Session.jsonl 是 append-only 事件日志**

→ Compaction **不改写磁盘**，只折叠内存中传给模型的 messages。`--resume` 看到的还是完整原文。

→ 反向证明：如果改写 jsonl，会同时损失取证能力和现场证据。

---

## 5. 多 Provider / 多模型

### 7 个 Provider

```
firstParty (Anthropic) │ Bedrock (AWS)  │ Vertex (Google Cloud)
Foundry                │ OpenAI         │ Gemini  │ Grok (xAI)
```

通过 **stream adapter 模式** 统一封装 — 核心稳定，每个 provider 一个适配器吸收差异。

### 三个独立的成本控制杠杆

| 杠杆 | 命令 | 做什么 | 何时用 |
|---|---|---|---|
| **Provider 选择** | `/login` | 切换到便宜的第三方 API（Gemini/Grok/OpenAI） | 长任务、低敏感场景 |
| **模型选择** | 启动参数 | 切到 1M context 型号（如 Gemini） | 大上下文任务 |
| **/poor 穷鬼模式** | `/poor` | 关掉后台 API 调用 | 全场景节流 |

`/poor` 具体跳过：
- `extract_memories`（自动抽取 memory）
- `prompt_suggestion`（输入框下方建议）
- `verification_agent`（任务完成后独立验证 fork）

→ 这三个都是**额外的 API 调用**，关了能省真金白银。

**组合使用**：成本敏感的大上下文任务 = `/poor` + 第三方 API + 1M 模型，三者正交叠加。

---

## 6. 高级运行模式

REPL 之外还有 7 种运行形态，每种解决不同问题：

| 模式 | 入口 | 解决什么问题 |
|---|---|---|
| **Daemon** | `claude daemon` | 常驻 supervisor + worker 池，避免每次启动开销 |
| **Remote Control Server** | `claude remote-control` / `rc` / `bridge` | 自托管远程服务器，浏览器/手机远程驱动 Claude Code，含 Web UI |
| **Computer Use MCP** | `--computer-use-mcp` | 让 Claude 控制本机桌面（截图 / 鼠键 / 应用管理） |
| **Claude in Chrome MCP** | `--claude-in-chrome-mcp` | 让 Claude 驱动 Chrome 操作网页 |
| **Voice Mode** | 启动 + 配置 | Push-to-Talk 语音输入（需 Anthropic OAuth） |
| **ACP** | `acp-link` 桥接 | IDE / 外部客户端通过协议接入（WebSocket → ACP agent） |
| **Worktree** | `EnterWorktree` 工具 | git worktree 隔离的实验环境 |

### 何时选哪个

- 想**长跑、省冷启动**：daemon
- 想**手机/远程**操作：RCS
- 想让 AI **替你点鼠标**：Computer Use（系统级）/ Chrome MCP（网页级）
- 想**说话写代码**：Voice
- 想让 **IDE 接入** Claude：ACP
- 想**并行试 3 个方案**：Worktree

---

## 7. 跨能力的 3 个迁移性洞察

这是 Claude Code 架构在多个功能上**反复出现的设计模式**：

### ① 核心稳定，变化关进盒子

- 7 个 provider → 同一个 `query()`，差异在 stream adapter 里
- 不同权限模式 → 同一个 turn loop，差异在传入的 `canUseTool` 实现里
- 19 个 feature flag → 编译期常量替换，运行时无 if/else

→ **不要加 flag 解决变体，要换实现**。

### ② Cache 顺序：稳定在前、变化在后

- system prompt：稳定指令在前，模型动态 inject 在后
- CLAUDE.md（注入 prefix）：cache 命中
- Auto memory（按需读尾部）：cache 不被打破

→ 写 CLAUDE.md / memory 时遵循这条，可以**最大化 prompt cache 命中率**。

### ③ 性能让位于原则、原则让位于极端性能

- AppState 用响应式（原则正确）
- Token 计数放 `bootstrap/state.ts` 单例（性能例外，因为每个 token 触发 re-render 太贵）
- jsonl 永远 append-only（原则不让步，否则失去取证）

---

## 8. 一页速查

```
INTERACTION    REPL │ -p (Headless) │ /plan │ auto-mode
TOOLS          File │ Bash │ Agent/Task │ Web │ MCP │ Cron │ Worktree
EXTEND         MCP │ Skills │ Hooks │ Slash │ Subagents
MEMORY         CLAUDE.md (注入) │ memory (按需读) │ session (append) │ compaction (内存折叠)
PROVIDERS      7 家 / /login / 1M context / /poor (省 token)
ADVANCED       daemon │ RCS │ Computer Use │ Chrome │ Voice │ ACP │ Worktree

省钱组合       /poor + 第三方 API + 1M 模型（三者正交）
长跑组合       daemon + Auto Mode + 模板任务
团队远程       RCS + Web UI + ACP（IDE 接入）
并行实验       Worktree × N + Agent fork
```

---

## 9. 给团队的上手建议

按"立刻有用 → 进阶"排序：

1. **先用熟 Plan Mode**：高风险变更前 `/plan`，让 Claude 先写方案，看完再让动手
2. **写一份 CLAUDE.md**：项目约定、测试命令、不可触碰的文件
3. **配 1-2 个 hook**：在 PreToolUse / PostToolUse 加 lint / typecheck 兜底
4. **试一次 Worktree**：让 Claude 在隔离分支上做实验，不污染主分支
5. **试一次 fork Agent**：把研究类任务丢给 fork，主对话保持简洁
6. **试一次 Auto Mode**：拿一个重复性任务（批量改 doc / 升级依赖）跑一晚
7. **配一台远程 RCS**：手机 / 出差也能用

---

## 10. 进一步学习

| 想深入 | 建议 |
|---|---|
| **架构原理** | 看 cli.tsx → main.tsx → REPL.tsx → QueryEngine → query.ts 这条链 |
| **Auto Memory 详解** | 单独一份指南：`.claude/skills/teach-me/records/auto-memory/auto-memory-guide.md` |
| **Hooks 配置** | settings.json + 各生命周期事件名 |
| **MCP 自建** | 用 mcp-builder skill 起步 |
| **RCS 部署** | `docs/features/remote-control-self-hosting.md` |
