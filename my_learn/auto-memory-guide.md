# Claude Code Auto Memory 详解

> 一份自包含的 auto memory 机制学习总结，可独立阅读、可分享给团队。
> 适用对象：Claude Code 使用者，已了解 CLAUDE.md 的人。
> 一句话定位：**memory 是 Claude 给"未来的自己"留的、跨会话生效的、按需读取的 .md 笔记。**

---

## 1. 一图看懂

```
┌─────────────────────────────────────────────────────────────┐
│                      持久化层级                              │
├──────────────┬─────────────┬───────────────┬────────────────┤
│  CLAUDE.md   │ Auto Memory │  TaskCreate   │  Plan 文件     │
├──────────────┼─────────────┼───────────────┼────────────────┤
│ 进 git       │ 不进 git    │ 本会话内      │ 本任务内       │
│ 团队共享     │ 私人        │ 多步进度      │ 实施路线       │
│ 注入 prefix  │ 按需读       │ 内存中状态    │ 临时文件       │
│ 项目级约定   │ 跨会话个人  │ 会话结束消失  │ 任务结束消失   │
└──────────────┴─────────────┴───────────────┴────────────────┘
                ▲
                │ 本文档聚焦这一层
                ▼
       ┌────────────────────────────────────────────┐
       │  ~/.claude/projects/<cwd-slug>/memory/     │
       │  ├── user_xxx.md          (你是谁)         │
       │  ├── feedback_xxx.md      (怎么干活)       │
       │  ├── project_xxx.md       (项目动机/状态)  │
       │  └── reference_xxx.md     (外部系统位置)   │
       └────────────────────────────────────────────┘
```

---

## 2. 存储位置

```
~/.claude/projects/-{cwd-slug}/memory/<name>.md
```

- `cwd-slug` = 当前工作目录路径替换 `/` 为 `-`
  - 例：`/Users/jia/Desktop/claude-code` → `-Users-jia-Desktop-claude-code`
- **按项目隔离**：换项目目录就是另一套 memory
- 不进 git，**只是你本机**

---

## 3. 文件结构

```markdown
---
name: 简短名
description: 一句话（决定未来检索时的相关性，越具体越好）
type: user | feedback | project | reference
---

{body 内容}
```

`description` 很关键 — 它是模型未来 grep 命中后判断 "要不要 Read 这个文件" 的依据。写得越具体越好。

---

## 4. 4 种类型的语义边界

| 类型 | 内容 | 例子 | 是否要 Why |
|---|---|---|---|
| **user** | 你是谁、你的角色/背景/个人偏好 | "Go 十年、第一次 React" / "金融背景偏向安全于性能" | 否 |
| **feedback** | 你对"怎么干活"的明确指导 | "不要 mock DB" / "别在结尾写总结" | **必需** |
| **project** | 项目状态 / 进行中的事 / 决策动机 / 截止时间 | "鉴权重写是合规驱动" / "3-5 后冻结合入" | **必需** |
| **reference** | 外部系统的指针 | "Bug 在 Linear INGEST" / "监控 grafana.internal/d/api" | 否 |

### 为什么 feedback / project 必须有 Why？

> 规则只能告诉你"做什么"，不能告诉你"何时不适用"。
>
> 例：规则 = "不 mock DB"。Why = "上季度 mock 测试 pass 但 prod 迁移失败"。
>
> → 一年后遇到不涉及迁移的纯单元测试，看到 Why 你才知道这条**还适不适用**。

---

## 5. body 模板（feedback / project）

```markdown
{规则或事实}

**Why:** {原因 — 通常是过往事件、强偏好、合规约束}
**How to apply:** {何时/何处适用、边界条件}
```

`user` 和 `reference` 是事实陈述，body 自由写即可。

---

## 6. 写入触发（两条并行）

| 触发 | 时机 | 谁决定 | 受 /poor 影响 |
|---|---|---|---|
| **直接 Write** | 模型在 turn 中察觉值得记 | 模型当场判断 | 否（已在 context，free） |
| **extract_memories** | 任务结束后台抽取 | 单独一次 API 调用专门提炼 | **是**（穷鬼模式跳过） |

→ 启用 `/poor` 后只剩主动 Write，不会有后台抽取。

---

## 7. 读取触发（按需，不预加载）

memory **不进** system prompt（这是和 CLAUDE.md 的关键区别）。

模型在以下情况主动 grep + Read：
- 用户提到过往工作 / 主动让 recall
- 当前问题"似乎相关"（模型自行判断 description）
- 用户**显式**说"忽略 memory" → 当作不存在

---

## 8. 不该写什么（即使用户让你记）

| 类别 | 例 | 为什么 |
|---|---|---|
| 可推导的项目约定 | "项目用 bun test" | package.json 看得到 |
| Git 中可查的事 | "上周提了 PR #123" | git log 即可 |
| 调试 fix 方案 | "这个 bug 修法是颠倒调用顺序" | fix 在代码里、context 在 commit |
| 当前会话的中间状态 | "正在等 review" | 用 Task / Plan，不是 memory |
| CLAUDE.md 已写过 | 重复内容 | 避免漂移 |
| 包含潜在 PII / 凭证 | tokens、密码 | 安全 |

**判断口诀**：
> "未来 Claude 想知道这个，是去 git/grep/读代码更准，还是去翻 memory 更准？"
>
> 前者 → 不存。

---

## 9. 陈旧检测（最重要的实践）

> **"memory 说 X 存在" ≠ "X 现在存在"。**

memory 是某时刻的快照，**不是当前真相**。

任何包含**文件路径 / 函数名 / flag 名**的 memory，被推荐前必须验证：

| memory 内容 | 验证动作 |
|---|---|
| 路径 | 检查文件存在 |
| 函数 / flag | grep |
| 即将基于推荐**行动**（不只是查历史） | 必须验证 |
| 询问"过去是什么样" | 不必验证（描述历史本身就是目的） |

---

## 10. 与其他持久化机制的边界

| 信息 | 应该存哪 | 理由 |
|---|---|---|
| "本项目不要用 emoji"（团队共享） | CLAUDE.md（项目级） | 进 git，团队所有人生效 |
| "我个人讨厌 Claude 写总结结尾" | CLAUDE.md（用户级 `~/.claude/CLAUDE.md`）或 feedback memory | 跨项目 + 个人 |
| "今天调下列 5 件事" | TaskCreate | 本会话生命周期 |
| "重构拆 3 个阶段" | Plan 文件 | 本任务实施路线 |
| "用户从 Go 转到 React" | user memory | 跨会话个人背景 |
| "Bug 在 Linear INGEST 跟踪" | reference memory | 外部系统位置 |
| "鉴权重写是合规驱动" | project memory | 动机不在代码里 |

---

## 11. 维护：/dream

`/dream` skill = 手动触发 memory 整理：
- 翻看所有 memory
- 合并重复
- 删除过时（验证 path/函数还在不在）
- 按主题归并

**建议节奏**：积累 5+ memory 后跑一次。

---

## 12. 三条最常踩的坑

### ❌ 把项目约定写进 memory
"项目用 bun test 不用 jest" — 这是**约定**，写到 CLAUDE.md。memory 是给**看代码看不出来的**信息留的。

### ❌ 写规则不写 Why
"不要 mock DB" — 一年后没人能判断这条还适不适用。**永远带 Why**。

### ❌ 把 memory 当成事实来源
memory 写了 `PaymentService.process` 函数 → 不代表它现在还在。**推荐前 grep**。

---

## 13. 实操建议

第一次给团队铺 memory，按这个顺序：

1. **先盘 CLAUDE.md** — 项目级约定先放这（团队共享）
2. **写一两条 user memory** — 自己的背景、技能栈分布
3. **每次 Claude 纠正你方向、或你纠正 Claude 时** — 写一条 feedback（带 Why）
4. **每次重要项目决策** — 写一条 project（带 Why + 截止时间）
5. **2-3 周后 `/dream`** — 看看哪些过时了

---

## 14. 一页速查

```
WHERE        ~/.claude/projects/-<cwd-slug>/memory/<name>.md
TYPES        user / feedback / project / reference
TEMPLATE     ---
             name / description / type
             ---
             {规则}
             **Why:**           ← feedback/project 必需
             **How to apply:**  ← feedback/project 必需

WRITE WHEN   - 跨会话还会用
             - 看代码推不出来
             - 不是临时状态

DON'T WRITE  - 可推导的约定 → CLAUDE.md
             - git 可查的    → 别存
             - fix recipe    → 在 commit 里
             - 临时进度      → Task / Plan

READ WHEN    - 模型按需 grep + Read（不预注入）

VERIFY       - 含 path/函数/flag 的 memory，行动前 grep

MAINTAIN     - 5+ 条后 /dream 一次
```
