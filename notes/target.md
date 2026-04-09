# 汇总版本

---

# 概念引入

### 大模型是什么

大语言模型（LLM）核心能力是**理解和生成自然语言**。它接收一段文本（prompt），输出一段续写文本（completion）。为了方便理解，我们可以认为它是一个**无状态的文本生成函数**。

关键局限：**无状态**（不记得上一次对话）、**无行动力**（只能输出文本，不能读文件、跑命令）、**上下文有限**（输入 token 有上限）。

---

### Agent 是什么

Agent 是在大模型之上增加了**感知-决策-行动**循环的系统。最经典的范式是 **ReAct（Reasoning + Acting）**：

```
用户输入 → 思考(Reasoning) → 行动(Acting) → 观察(Observation) → 再思考 → ... → 输出答案
```

| ReAct 步骤 | 含义 |
|------------|------|
| **Reasoning（思考）** | 模型分析当前状态，决定下一步做什么 |
| **Acting（行动）** | 调用外部工具（API、文件操作、命令执行等） |
| **Observation（观察）** | 获取工具返回的真实结果，作为下一轮思考的输入 |

与LLM的区别：**有工具**（能调用外部 API、读写文件、执行命令）、**有循环**（多步推理）、**有决策权**（AI 自己决定用哪个工具、传什么参数、何时停止）。

基础的 ReAct(思考→行动→观察)解决了 LLM 无状态无行动力上下文隔离的核心问题，现在假设我们拿到了和 Claude Code 完全相同的 system prompt 和工具集，我们这个简单的 agent 能够像 cc 一样优秀的工作吗？很显然是不能的，因为我们很快会遇到这些问题：API 报错直接崩溃、对话太长 token 超限直接崩溃、AI 执行了危险命令如`rm -rf /`把我们库给删除了、需要同时改 5 个文件只能一个一个串行删除等等问题。

和 Claude Code 一样的 prompt、一样的工具弄出来的 agent，最终体验效果和 cc非常大？差距在哪？差就差在：
1. **prompt 背后有一整套编排机制**
2. **工具背后有一整条治理流水线**
3. **Agent 背后有一套分工和调度系统**
4. **生态扩展背后有一套让模型"知道自己能干什么"的感知通道**

这些东西加在一起，才构成了你用 Claude Code 时感受到的那个"稳"。这个稳不是来自某段神奇文案，是来自工程。

---

# 第二步：Prompt 编排机制
不是靠一段写死的 prompt，而是靠一套**动态组装、预算管理、自动压缩**的编排机制。

```
┌─────────────────────────────────────────────────────────────┐
│                    Prompt 编排全景                           │
├─────────────────────────────────────────────────────────────┤
│  1. CLAUDE.md         ← 项目级知识注入（最低成本定制）         │
│  2. 环境信息注入       ← Git 状态、日期、模型信息              │
│  3. System Prompt     ← 动态组装 + 缓存分块                   │
│  4. Token 预算        ← 200K 不是全部，预留管理               │
│  5. 四层压缩          ← 时间/缓存编辑/记忆/摘要递进            │
│  6. 项目记忆          ← 跨对话持久化                          │
└─────────────────────────────────────────────────────────────┘
```

## 1. CLAUDE.md：项目级知识注入（最重要）

在项目根目录放一个 `CLAUDE.md` 文件，就能让 AI "理解"你的项目。内容建议：
- 项目概述、技术栈
- 开发约定（代码风格、命名规范）
- 常用命令（构建、测试、部署）
- 已知的坑、特殊配置

系统自动发现并合并多级 CLAUDE.md：

```
~/.claude/CLAUDE.md          ← 用户全局（个人偏好）
  └── /project/CLAUDE.md     ← 项目根目录（团队共享）
        └── /src/CLAUDE.md   ← 子目录（模块特定）
```

**这是编排机制中最直接、最实用的部分**——写一个文件，AI 就能获得项目上下文。

---

## 2. 环境信息注入：AI 的"感知器官"

System Prompt 中的环境信息让 AI "知道"当前工作状态。这些信息通过多个函数分层注入：

### 用户上下文（`getUserContext`）

```
用户上下文注入
├── claudeMd: CLAUDE.md 内容
│   ├── 多级合并：从 CWD 向上遍历目录树
│   ├── 层级：~/.claude/CLAUDE.md → /project/CLAUDE.md → /src/CLAUDE.md
│   └── 禁用：CLAUDE_CODE_DISABLE_CLAUDE_MDS=1 或 --bare 模式
└── currentDate: 当前日期（Today's date is YYYY-MM-DD）
```

### 系统上下文（`getSystemContext`）

```
系统上下文注入
├── gitStatus: Git 状态快照
│   ├── 当前分支名
│   ├── 主分支名（用于 PR）
│   ├── Git 用户名
│   ├── 未提交变更列表（截断至 2000 字符）
│   └── 最近 5 条 commit（oneline 格式）
└── cacheBreaker: 调试用缓存破坏标记（Feature-gated: BREAK_CACHE_COMMAND）
```

### 环境信息（`computeSimpleEnvInfo`）

```
环境信息注入
├── 工作目录
│   ├── Primary working directory: /path/to/project
│   └── worktree 检测："This is a git worktree..."
├── Git 仓库状态
│   └── Is a git repository: Yes/No
├── 平台信息
│   ├── Platform: darwin / linux / win32
│   ├── Shell: zsh / bash
│   └── OS Version: Darwin 25.3.0 / Linux 6.6.4
├── 模型信息
│   ├── 模型名称：You are powered by the model named Claude Opus 4.6
│   ├── 知识截止日期：Assistant knowledge cutoff is August 2025
│   └── 模型 ID：claude-opus-4-6
└── Claude Code 可用性
    └── CLI / Desktop App / Web App / IDE Extensions
```

### 知识截止日期映射

| 模型 | 知识截止 |
|-----|---------|
| `claude-sonnet-4-6` | August 2025 |
| `claude-opus-4-6` / `claude-opus-4-5` | May 2025 |
| `claude-haiku-4-*` | February 2025 |
| `claude-opus-4` / `claude-sonnet-4` | January 2025 |

---

## 3. System Prompt：AI 的"工作记忆"

System Prompt 是每次 API 调用都发送的系统级指令，告诉 AI：
- 它是谁
- 能用什么工具
- 当前项目状态（git 分支、日期等）
- 项目知识（CLAUDE.md 内容）

**关键设计**：System Prompt 是 `string[]` 数组而非单个字符串，这是为了缓存分块——不变的部分可以复用，任何字符变化不会导致整个缓存失效。

**动态组装**：每次对话都重新构建，根据当前模型能力、MCP 连接状态动态调整。

### System Prompt 完整结构

```
┌─────────────────────────────────────────────────────────────┐
│ 静态区（scope: 'global' 可跨组织缓存）                         │
├─────────────────────────────────────────────────────────────┤
│ 1. Intro Section                                            │
│    - 身份声明："You are Claude Code..."                     │
│    - 网络安全提示：不生成/猜测 URL                            │
├─────────────────────────────────────────────────────────────┤
│ 2. System Section                                           │
│    - 工具执行模式：用户选择的权限模式                          │
│    - Hook 机制：用户可配置的事件响应                          │
│    - 自动压缩提示："对话有无限上下文"                          │
├─────────────────────────────────────────────────────────────┤
│ 3. Doing Tasks Section                                      │
│    - 任务执行规范：先读再改、不过度工程                        │
│    - 代码风格指导：不添加不必要的注释/抽象                     │
│    - 错误处理策略：诊断后修复，不盲目重试                      │
│    - 安全意识：避免 OWASP Top 10 漏洞                        │
├─────────────────────────────────────────────────────────────┤
│ 4. Actions Section                                          │
│    - 风险评估：破坏性操作、不可逆操作、共享状态操作            │
│    - 用户确认场景：删除、强制推送、发送消息等                  │
├─────────────────────────────────────────────────────────────┤
│ 5. Using Your Tools Section                                 │
│    - 工具优先级：Read/Edit/Write > Bash                      │
│    - 并行策略：独立调用并行，依赖调用串行                      │
├─────────────────────────────────────────────────────────────┤
│ 6. Tone and Style Section                                   │
│    - 输出风格：简洁、直接、无填充                              │
│    - 引用格式：file_path:line_number                         │
├─────────────────────────────────────────────────────────────┤
│ 7. Output Efficiency Section                                │
│    - 输出效率：先说结论，再说推理                              │
│    - 长度限制：工具调用之间 ≤25 词，最终响应 ≤100 词           │
├─────────────────────────────────────────────────────────────┤
│ === SYSTEM_PROMPT_DYNAMIC_BOUNDARY === （不发送 API）        │
├─────────────────────────────────────────────────────────────┤
│ 动态区（每次会话不同）                                         │
├─────────────────────────────────────────────────────────────┤
│ 8. Session-Specific Guidance                               │
│    - AskUserQuestion 使用场景                               │
│    - AgentTool 使用指南（fork vs subagent）                  │
│    - 技能调用方式（/斜杠命令）                                │
├─────────────────────────────────────────────────────────────┤
│ 9. Memory Section                                           │
│    - 记忆系统使用指南                                         │
│    - MEMORY.md 内容（如有）                                  │
├─────────────────────────────────────────────────────────────┤
│ 10. Language Section（如配置）                               │
│     - 语言偏好："Always respond in Chinese"                  │
├─────────────────────────────────────────────────────────────┤
│ 11. Output Style Section（如配置）                           │
│     - 自定义输出风格（如 "technical_writer"）                  │
├─────────────────────────────────────────────────────────────┤
│ 12. MCP Instructions（如有连接）                             │
│     - 每个 MCP Server 的使用说明                             │
├─────────────────────────────────────────────────────────────┤
│ 13. Scratchpad Instructions（如启用）                        │
│     - 临时文件目录使用指南                                    │
├─────────────────────────────────────────────────────────────┤
│ 14. Function Result Clearing（如启用 CACHED_MICROCOMPACT）   │
│     - "旧工具结果会被自动清除"                                 │
├─────────────────────────────────────────────────────────────┤
│ 15. Token Budget（如启用）                                  │
│     - token 预算目标指导                                     │
├─────────────────────────────────────────────────────────────┤
│ 16. Environment Info                                        │
│     - 工作目录、Git 状态、平台信息、模型信息                   │
└─────────────────────────────────────────────────────────────┘
```

### 动态组装三阶段管道

```
getSystemPrompt()                    // 组装内容
  ├── 判断 PROACTIVE 模式 → 返回简化版
  ├── 判断普通模式 → 组装完整结构
  └── resolveSystemPromptSections() // 并行解析动态 section
         ↓
buildSystemPromptBlocks()           // 分块 + cache_control 标记
  ├── splitSysPromptPrefix()       // 按 BOUNDARY 切分
  └── 为每个块添加 cache_control
         ↓
API 请求                             // 发送给 Claude API
```


## 4. Token 预算：200K 不是全部

Claude 默认上下文窗口 200K tokens，但实际可用空间远小于此：

```
上下文窗口（200K）
├── System Prompt（~15-25K）
├── 工具定义（~10-20K）
├── 输出预留（~8K-64K）
└── 剩余：对话历史空间（随对话增长）
```

### 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | 自动压缩触发缓冲 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 警告阈值缓冲 |
| `ESCALATED_MAX_TOKENS` | 64,000 | 输出截断恢复时的上限 |

当对话历史占用接近上限时，系统会**自动压缩**。

---

## 5. 压缩机制：四层递进策略

| 层级 | 触发条件 | 需要调 API | 效果 |
|------|---------|:---:|------|
| **时间触发 MicroCompact** | 距上次 assistant 消息超过阈值（如 5 分钟） | 否 | 清除旧工具输出（缓存已冷） |
| **Cached MicroCompact** | 工具输出数量超过阈值 | 否 | 通过 cache_edit 删除，不破坏缓存前缀 |
| **Session Memory 压缩** | 自动触发（实验性） | 否 | 用已有记忆替换旧对话 |
| **传统摘要压缩** | 手动 `/compact` 或兜底 | 是 | AI 生成对话摘要 |

### 第一层：时间触发的 MicroCompact

当距离上次 assistant 消息的时间超过配置的阈值（如 5 分钟），意味着服务器端缓存已经过期。此时：
- 直接修改消息内容，将旧工具输出替换为 `[Old tool result content cleared]`
- 保留最近 N 个工具结果（默认配置）
- 重置 Cached MicroCompact 的状态（因为缓存已冷）

**为什么时间触发不用 cache_edit**：cache_edit 假设缓存是热的，而我们刚确认它是冷的，直接修改内容即可。

### 第二层：Cached MicroCompact（Cache Editing API）

这是最精巧的一层：当工具输出数量超过阈值时，通过 `cache_edit` API 在**不破坏缓存前缀**的情况下删除工具结果。

```
传统方式：修改消息内容 → 缓存前缀失效 → 整个 System Prompt 重写（~20K tokens）
Cache Editing：发送 cache_edit 指令 → 服务器端删除指定 tool_result → 缓存前缀保持命中
```

**关键洞察**：这节省了大量 token 成本，因为 System Prompt 前缀（~15K tokens）可以继续复用缓存。

**白名单工具**（可被压缩）：
- `Read`（文件读取）
- `Bash`（命令输出）
- `Grep`（搜索结果）
- `Glob`（文件列表）
- `WebSearch` / `WebFetch`
- `FileEdit` / `FileWrite`

### 第三层：Session Memory 压缩

不需要调用摘要模型，直接使用已经提取好的 Session Memory 作为对话摘要。

**保留窗口计算**：
- 至少 10,000 token（有上下文深度）
- 至少 5 条包含文本的消息（有对话连续性）
- 最多 40,000 token（不会太大又触发下一次压缩）

**工具对完整性保护**：API 要求每个 `tool_result` 都有对应的 `tool_use`，系统确保压缩边界不切在 `tool_result` 消息处。

### 第四层：传统 API 摘要

**压缩前处理**：
- 图片被替换为 `[image]` 标记
- 移除会被重新注入的附件（skill_discovery、skill_listing）

**压缩后重新注入**（50K token 预算）：

| 重新注入内容 | 预算 |
|-------------|------|
| 最近读取的文件（最多 5 个） | 每个 5K token |
| 已激活的技能指令 | 总计 25K token |
| CLAUDE.md 内容 | 全量 |

**CompactBoundary**：每次压缩后，系统在消息流中插入一条边界标记，后续操作只处理最后一条 boundary 之后的消息。


---

## 6. 项目记忆：跨对话持久化

记忆系统存储在文件中，路径：

```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md              ← 入口索引（每次对话加载）
├── user_role.md           ← 用户角色/偏好
├── feedback_testing.md    ← 用户纠正
├── project_xxx.md         ← 项目上下文
└── reference_xxx.md       ← 外部系统指针
```

**MEMORY.md 索引限制**：200 行 AND 25KB（双重上限）

### 四类型记忆

| 类型 | 内容 | 触发场景 |
|-----|------|---------|
| **user** | 用户角色、技术背景 | "我是数据科学家" |
| **feedback** | 用户纠正/确认 | "别 mock 数据库" |
| **project** | 非代码可推导的上下文 | "周四开始合并冻结" |
| **reference** | 外部系统指针 | "bug 在 Linear INGEST 项目" |

**核心约束**：只存储**无法从当前项目状态推导**的信息。代码架构、文件路径、git 历史都可以实时获取，不需要记忆。

### 记忆文件格式

```markdown
---
name: 记忆名称
description: 一行描述（用于判断相关性）
type: user | feedback | project | reference
---

{{记忆内容}}
```

---

## 源码入口索引

| 主题 | 说明 | 入口文件 |
|-----|------|---------|
| **System Prompt 组装** | 静态区/动态区划分、Section 注册 | `src/constants/prompts.ts:445` (`getSystemPrompt`) |
| **缓存分块策略** | System Prompt 如何分块、标记 `cache_control` | `src/services/api/claude.ts:3255` (`buildSystemPromptBlocks`) |
| **CLAUDE.md 加载** | 多级合并、目录遍历 | `src/utils/claudemd.ts` (`getClaudeMds`) |
| **上下文构建** | Git 状态、日期、CLAUDE.md 注入 | `src/context.ts` |
| **Token 计数** | 近似估算 vs 精确计数 | `src/utils/tokens.ts`、`src/services/tokenEstimation.ts` |
| **自动压缩触发** | 阈值判断、逃逸条件 | `src/services/compact/autoCompact.ts` |
| **MicroCompact** | 工具输出清理、时间触发、Cache Editing | `src/services/compact/microCompact.ts` |
| **传统摘要压缩** | 图片剥离、摘要生成、上下文恢复 | `src/services/compact/compact.ts` |
| **Session Memory 压缩** | 保留窗口计算、工具对完整性保护 | `src/services/compact/sessionMemoryCompact.ts` |
| **Reactive Compact** | prompt_too_long 时的即时压缩 | `src/services/compact/reactiveCompact.ts` |
| **Compact Boundary** | 压缩边界标记、消息链重建 | `src/utils/messages.ts` (`createCompactBoundaryMessage`) |
| **记忆系统** | 四类型分类、行为指导、注入链路 | `src/memdir/memdir.ts`、`src/memdir/memoryTypes.ts` |
| **记忆召回** | Sonnet 侧查询、相关性筛选 | `src/memdir/findRelevantMemories.ts` |

> **详见**：[Prompt编排机制.md](./Prompt编排机制.md)



# 第三步：工具治理流水线
不是简单的"调用工具"，而是经过**分层注册、权限检查、输出预算、并发控制**的完整流水线。

## 工具种类

Claude Code 的所有工具按四个层级组装：

```
固定工具（始终可用）:
  AgentTool, BashTool, FileReadTool, FileEditTool, FileWriteTool,
  NotebookEditTool, WebFetchTool, WebSearchTool, TodoWriteTool,
  AskUserQuestionTool, SkillTool, EnterPlanModeTool, ExitPlanModeV2Tool,
  TaskOutputTool, BriefTool, ListMcpResourcesTool, ReadMcpResourceTool

条件工具（运行时检查）:
  ← hasEmbeddedSearchTools()  ? []       : [GlobTool, GrepTool]
  ← isTodoV2Enabled()         ? V2 Tasks  : []
  ← isWorktreeModeEnabled()   ? Worktree  : []
  ← isAgentSwarmsEnabled()    ? Teams     : []
  ← isToolSearchEnabled()     ? ToolSearch: []
  ← isPowerShellToolEnabled() ? PowerShell: []

Feature-flag 工具:
  ← feature('COORDINATOR_MODE') ? [coordinatorMode tools]
  ← feature('KAIROS')           ? [SleepTool, SendUserFileTool, ...]
  ← feature('WEB_BROWSER_TOOL') ? [WebBrowserTool]
  ← feature('HISTORY_SNIP')     ? [SnipTool]

Ant-only 工具:
  ← process.env.USER_TYPE === 'ant' ? [REPLTool, ConfigTool, TungstenTool]
```

**关键设计**：工具列表在每次 API 调用时组装（而非全局缓存），因为运行时状态可能变化。



## 使用链路


```
┌─────────────────────────────────────────────────────────────┐
│                    System Prompt 组成                        │
├─────────────────────────────────────────────────────────────┤
│ 静态区（可缓存）                                              │
├─────────────────────────────────────────────────────────────┤
│ 动态区（每次重建）                                            │
│   ├── 工具列表定义 ←───── getTools() 返回，占 10-20K tokens    │
│   ├── Tool.prompt() ←──── 每个工具的使用指南                   │
│   └── 当前状态（git/日期/CLAUDE.md）                           │
└─────────────────────────────────────────────────────────────┘
```

```
AI 发出 tool_use（name + input）
        ↓
    参数校验（validateInput）
        ↓ 失败 → 返回错误
    权限检查（checkPermissions + 弹窗确认）
        ↓ 拒绝 → 返回拒绝原因
    执行操作（call）
        ↓
    返回 ToolResult
        ↓
    追加到对话历史 → 进入下一轮迭代
```


# 第四步：Agent 的分工和调度系统

> **分层分工**：五层架构各司其职，QueryEngine 管会话，query.ts 管循环，子 Agent 处理子任务，多 Agent 协作处理大规模任务。
> **协调调度**：协调 Prompt 编排和工具治理，通过状态机循环完成任务，遇到问题协调两个系统恢复。


## 一、分工体系：五层架构

```
┌─────────────────────────────────────────────────────────────┐
│ 第 1 层：交互层（REPL）                                       │
│   • 接收用户输入                                              │
│   • 展示消息流                                                │
│   • 处理键盘事件（ESC 中断、Tab 补全）                         │
│   入口：src/screens/REPL.tsx                                 │
├─────────────────────────────────────────────────────────────┤
│ 第 2 层：编排层（QueryEngine）                                │
│   • 管理多轮会话状态                                          │
│   • 构建 System Prompt（调用 Prompt 编排）                    │
│   • 持久化对话历史                                            │
│   • 追踪 token 消耗和成本                                     │
│   入口：src/QueryEngine.ts                                   │
├─────────────────────────────────────────────────────────────┤
│ 第 3 层：循环层（query.ts）                                   │
│   • 执行 Agentic Loop（状态机）                               │
│   • 协调工具执行（调用工具治理）                               │
│   • 错误恢复和重试                                            │
│   • 触发压缩（调用 Prompt 编排）                              │
│   入口：src/query.ts                                         │
├─────────────────────────────────────────────────────────────┤
│ 第 4 层：工具层（tools.ts）                                   │
│   • 组装工具列表                                              │
│   • 权限检查                                                  │
│   • 执行具体工具                                              │
│   入口：src/tools.ts, src/Tool.ts                           │
├─────────────────────────────────────────────────────────────┤
│ 第 5 层：通信层（API）                                        │
│   • 流式请求/响应                                             │
│   • 多 Provider 支持                                          │
│   • 错误重试                                                  │
│   入口：src/services/api/claude.ts                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心分工：QueryEngine vs query.ts

这是理解调度系统的**关键边界**：

```
┌─────────────────────────────────────────────────────────────┐
│                     QueryEngine                              │
│                    （会话编排器）                              │
│                                                              │
│  职责：管理"跨 turn"的会话状态                                 │
│  ─────────────────────────────────────────────────────────  │
│  • mutableMessages：完整对话历史，跨 turn 累积                │
│  • readFileState：已读文件缓存                               │
│  • totalUsage：累计 token 消耗                               │
│  • permissionDenials：权限拒绝记录                           │
│  • abortController：会话级中断控制                            │
│                                                              │
│  核心方法：submitMessage() → async *Generator               │
│  • 解析模型参数（支持模型热切换）                              │
│  • 调用 Prompt 编排构建 System Prompt                        │
│  • 调用 query() 进入 Agentic Loop                            │
│  • 持久化对话到 JSONL 文件                                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ 调用
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                       query.ts                               │
│                    （循环执行器）                              │
│                                                              │
│  职责：执行"单次请求"的 ReAct 循环                             │
│  ─────────────────────────────────────────────────────────  │
│  • State：当前迭代状态                                        │
│  • transition：状态转换原因                                   │
│  • StreamingToolExecutor：工具并发执行                        │
│  • 错误恢复逻辑                                               │
│                                                              │
│  核心方法：query() → async *Generator                        │
│  • while(true) 状态机循环                                    │
│  • 预处理 → API 调用 → 工具执行 → 终止/继续判定               │
│  • 调用工具治理获取工具列表                                   │
│  • 调用 Prompt 编排触发压缩                                   │
└─────────────────────────────────────────────────────────────┘
```

### 边界对比表

| 维度 | QueryEngine | query.ts |
|------|-------------|----------|
| **作用域** | 跨 turn（整个会话） | 单次请求（单轮对话） |
| **状态管理** | 持久化状态 | 临时迭代状态 |
| **Prompt 编排** | **构建** System Prompt | **使用** System Prompt |
| **工具治理** | 获取工具列表传入 | 执行工具、处理结果 |
| **压缩** | 记录压缩边界 | 触发压缩决策 |
| **错误恢复** | 记录错误历史 | 执行恢复逻辑 |

---

## 三、子 Agent 分工

当任务复杂时，主 Agent 可以派生子 Agent 分工处理：

### 两种派生方式

| 方式 | 特点 | 适用场景 |
|------|------|---------|
| **命名 Agent** | 独立 System Prompt、独立工具池 | 专用任务（code-reviewer、test-runner） |
| **Fork 子进程** | 继承父 Prompt Cache、共享工具池 | 并行子任务 |

### AgentTool 调用流程

```
主 Agent 决定派生子任务
        ↓
    调用 AgentTool
        ↓
┌───────────────────────────────┐
│ AgentTool 执行：               │
│ 1. 选择子 Agent 类型           │
│ 2. 构建 Prompt（可定制）       │
│ 3. 创建独立工具池（可裁剪）    │
│ 4. 调用 query() 执行          │
│ 5. 收集结果返回主 Agent       │
└───────────────────────────────┘
        ↓
    子 Agent 结果注入主对话
        ↓
    主 Agent 继续工作
```

### Worktree 隔离

子 Agent 可在独立的 git worktree 中工作，隔离文件修改：

```
主仓库（main 分支）
    │
    └── .claude/worktrees/feature-x/
            │
            └── 子 Agent 在此工作
                • 独立的文件系统
                • 独立的 git 状态
                • 完成后可合并或丢弃
```

## 四、多 Agent 协作分工

### 模式 1：Coordinator Mode（星型编排）

```
              ┌─────────────┐
              │ Coordinator │ ← 决策者、分配者
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
    Worker A     Worker B     Worker C
    （代码审查）  （测试运行）  （文档生成）
```

**职责分工**：
- **Coordinator**：分析需求、分解任务、分配 Worker、收集结果、综合验证
- **Worker**：执行具体子任务，通过 `<task-notification>` 汇报完成

**通信机制**：
- **Scratchpad**：跨 Worker 共享知识库
- **Task Notification**：Worker 完成 → XML 格式通知 Coordinator

---

### 模式 2：Agent Swarms（蜂群协作）

```
Agent 1 ←→ Agent 2 ←→ Agent 3
    ↑            ↑            ↑
    └────────────┴────────────┘
              任务队列
           （竞争认领）
```

**职责分工**：
- 所有 Agent **对等**，无主从关系
- 任务队列竞争认领
- 文件锁防止冲突

**适用场景**：大规模并行任务（如批量修复 100 个 lint 错误）

---

## 五、调度系统：Agentic Loop

在分工体系下，`query.ts` 负责执行 Agentic Loop：

### 四阶段循环

```
┌─────────────────────────────────────────────────────────────┐
│ 阶段 1：上下文预处理                                          │
│   applyToolResultBudget → microcompact → autocompact        │
│   （调用 Prompt 编排执行压缩）                                 │
├─────────────────────────────────────────────────────────────┤
│ 阶段 2：API 调用                                              │
│   callModel() → 流式响应                                     │
│   （Prompt 编排提供 System Prompt，工具治理提供 Tools）        │
├─────────────────────────────────────────────────────────────┤
│ 阶段 3：工具执行                                              │
│   StreamingToolExecutor / runTools()                         │
│   （工具治理提供并发安全判定和权限检查）                        │
├─────────────────────────────────────────────────────────────┤
│ 阶段 4：终止/继续判定                                         │
│   • 终止：completed / aborted_streaming / blocking_limit    │
│   • 继续：正常循环 / 错误恢复                                 │
└─────────────────────────────────────────────────────────────┘
```

### 终止与继续条件

**7 种终止条件**：

| 终止原因 | 说明 |
|----------|------|
| `completed` | AI 无 tool_use，任务完成 |
| `aborted_streaming` | 用户按 ESC 中断 |
| `blocking_limit` | Token 硬限制 |
| `model_error` | API 异常 |
| `prompt_too_long` | 压缩失败 |
| `image_error` | 图片处理错误 |
| `stop_hook_prevented` | Hook 阻止 |

**4 种继续条件**：

| 继续条件 | 恢复机制 |
|---------|---------|
| 正常工具循环 | 无 |
| `max_output_tokens` | 提升上限，注入恢复消息 |
| `prompt_too_long` | 即时压缩 |
| `stop_hook_blocking` | 注入错误消息重新思考 |

---
## 九、源码入口

| 主题 | 入口文件 | 职责 |
|-----|---------|------|
| **REPL** | `src/screens/REPL.tsx` | 交互层 |
| **QueryEngine** | `src/QueryEngine.ts` | 编排层 |
| **Agentic Loop** | `src/query.ts` | 循环层 |
| **工具注册** | `src/tools.ts` | 工具层 |
| **子 Agent** | `src/tools/AgentTool/` | Agent 派生 |
| **多 Agent 协作** | `src/teams/` | 协作模式 |


---

## 总结

> **Prompt 编排负责"知道什么"**（上下文、记忆、压缩）
> **工具治理负责"能做什么"**（工具、权限、安全）
> **调度系统负责"怎么做事"**（分工、状态机、错误恢复）

### Claude Code 是什么

Claude Code 自称 **agentic coding system**，而非 AI assistant 或 copilot。这个定位包含三个关键词：

| 关键词 | 含义 |
|--------|------|
| **Terminal-native** | 原生 CLI 应用，不是 IDE 插件、不是 Web 界面、不是 API wrapper |
| **Agentic** | AI 自主决策工具调用链，不是"一问一答"的聊天模式 |
| **Coding system** | 面向软件工程全流程，不是通用问答工具 |

它与简单 Agent 的区别在于：它是一个**生产级的 ReAct 实现**，在基础的"思考→行动→观察"循环之上，增加了：
- **自愈状态机**：API 错误自动重试/降级、token 超限自动压缩、用户中断优雅处理
- **分层上下文工程**：System Prompt → 对话历史 → Compaction → Memory 文件，营造记忆幻觉
- **双重安全模型**：应用层权限（白/黑名单）+ OS 层沙箱隔离
- **会话管理**：持久化、恢复、成本追踪

核心差异在于：它拥有**完整的 shell 访问权**——可以做任何你在终端里能做的事。这既是最大的能力，也是最大的风险。

---

### 架构全景：五层架构

| 层次 | 职责 | 入口源码 | 关键词 |
|------|------|----------|--------|
| **交互层** | 终端 UI、用户输入、消息展示 | `src/screens/REPL.tsx` | React/Ink、PromptInput |
| **编排层** | 多轮对话、会话持久化、成本追踪 | `src/QueryEngine.ts` | QueryEngine、transcript |
| **核心循环层** | 发请求 → 拿响应 → 执行工具 → 循环 | `src/query.ts` | Agentic Loop、State |
| **工具层** | AI 的"双手"——读写文件、执行命令（50+ 工具） | `src/tools.ts` → `src/Tool.ts` | Tool 接口、MCP |
| **通信层** | 与 Claude API 的流式通信 | `src/services/api/claude.ts` | Streaming、Provider |

---

Claude Code 的"稳"来自四个系统的协同：

| 系统 | 解决的问题 | 核心机制 |
|-----|-----------|---------|
| **Prompt 编排机制** | AI 怎么理解上下文？ | CLAUDE.md + 动态组装 + Token 预算 + 自动压缩 |
| **工具治理流水线** | 工具怎么安全执行？ | 分层注册 + 权限分级 + 命令解析 + 输出预算 + 后台化 |
| **Agent 分工调度** | Agent 怎么运转？ | Agentic Loop 状态机 + QueryEngine 编排 |
| **生态扩展感知通道** | 模型怎么知道能干什么？ | Tool.prompt() + ToolSearch + 技能 + MCP |

**这就是为什么"拿到和 Claude Code 完全相同的 prompt 和工具集，弄出来的 agent 体验差很多"**——你只拿到了静态定义，没拿到这套动态系统。
> 在大模型的文本生成能力之上，构建了一套完整的 **ReAct 循环**（思考→行动→观察），配合五层架构（交互→编排→循环→工具→通信）、自愈状态机、双重安全模型、分层上下文工程，使 AI 能够**自主地**在真实开发环境中完成软件工程任务。