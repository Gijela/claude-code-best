# 零基础 学CC

---

## 第一步：建立直觉

### 大模型是什么

大语言模型（LLM）核心能力是**理解和生成自然语言**。它接收一段文本（prompt），输出一段续写文本（completion）。为了方便理解，我们可以认为它是一个**无状态的文本生成函数**。

关键局限：
- **无状态**：不记得上一次对话
- **无行动力**：只能输出文本，不能读文件、跑命令
- **上下文有限**：输入 token 有上限

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

和 Claude Code 一样的 prompt、一样的工具弄出来的 agent，最终体验效果和 cc非常大？差距在哪？差在 prompt 背后有一整套编排机制，工具背后有一整条治理流水线，Agent 背后有一套分工和调度系统，生态扩展背后有一套让模型"知道自己能干什么"的感知通道。这些东西加在一起，才构成了你用 Claude Code 时感受到的那个"稳"。这个稳不是来自某段神奇文案，是来自工程。

---

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

## 第三步：对话引擎

### Agentic Loop = 生产级 ReAct

Claude Code 的核心循环本质上就是 **ReAct 模式的生产级实现**。

#### 四阶段结构

每次 `while(true)` 迭代包含四个阶段：

```
┌─────────────────────────────────────────────────────────────┐
│  ① 上下文预处理管道                                          │
│     applyToolResultBudget → snipCompact → microcompact      │
│     → contextCollapse → autocompact                         │
├─────────────────────────────────────────────────────────────┤
│  ② 流式 API 调用                                             │
│     deps.callModel() → AsyncGenerator<StreamEvent>          │
│     StreamingToolExecutor 并行执行工具                       │
├─────────────────────────────────────────────────────────────┤
│  ③ 工具执行                                                  │
│     StreamingToolExecutor（并行）或 runTools（串行）         │
├─────────────────────────────────────────────────────────────┤
│  ④ 终止/继续判定                                             │
│     needsFollowUp ? continue : return { reason }            │
└─────────────────────────────────────────────────────────────┘
```

#### 终止与继续条件

**7 种终止条件**：`completed`（无工具调用）、`blocking_limit`（token 硬限制）、`aborted_streaming`（用户中断）、`model_error`、`prompt_too_long`、`image_error`、`stop_hook_prevented`

**4 种继续条件**：
- 正常工具循环（有 tool_use → 执行 → continue）
- `max_output_tokens` 恢复（提升输出上限，注入恢复消息重试）
- Prompt-Too-Long 恢复（Context Collapse Drain / Reactive Compact）
- Stop Hook 阻塞重试（注入错误消息强制 AI 重新思考）

---

### QueryEngine：会话编排器

`QueryEngine`（`src/QueryEngine.ts`）在单轮 Agentic Loop 之上管理多轮会话：

```
QueryEngine 内部状态
├── mutableMessages: Message[]           ← 完整对话历史，跨 turn 累积
├── readFileState: FileStateCache        ← 已读文件缓存
├── totalUsage: NonNullableUsage         ← 累计 token 消耗
├── permissionDenials: SDKPermissionDenial[]  ← 权限拒绝记录
└── abortController: AbortController     ← 会话级中断控制
```

核心方法 `submitMessage()` 是 `async *Generator`，逐步 yield 消息让 REPL/SDK 实时展示进度。

---

### 会话持久化

```
~/.claude/projects/<project-hash>/<session-id>.jsonl
```

JSONL 格式支持追加写入，`--resume` 从历史消息恢复会话。包含成本追踪（`totalCostUSD`、按模型分拆用量）和文件快照（绑定到 message.id，支持 `--rewind-files` 精确回滚）。

---

## 第四步：上下文工程

**核心问题**：LLM 是无状态的，上下文窗口有限。Claude Code 如何让 AI "记住"项目信息、管理有限 token？

---

### CLAUDE.md：项目级知识注入（最重要）

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

**这是上下文工程中最直接、最实用的机制**——写一个文件，AI 就能获得项目上下文。

---

### System Prompt：AI 的"工作记忆"

System Prompt 是每次 API 调用都发送的系统级指令，告诉 AI：
- 它是谁
- 能用什么工具
- 当前项目状态（git 分支、日期等）
- 项目知识（CLAUDE.md 内容）

**关键设计**：System Prompt 是 `string[]` 数组而非单个字符串，这是为了缓存分块——不变的部分可以复用，任何字符变化不会导致整个缓存失效。

---

### Token 预算：200K 不是全部

Claude 默认上下文窗口 200K tokens，但实际可用空间远小于此：

```
上下文窗口（200K）
├── System Prompt（~15-25K）
├── 工具定义（~10-20K）
├── 输出预留（~8K-64K）
└── 剩余：对话历史空间（随对话增长）
```

当对话历史占用接近上限时，系统会**自动压缩**。

---

### 压缩机制：三层递进策略

| 层级 | 触发条件 | 需要调 API | 效果 |
|------|---------|:---:|------|
| **MicroCompact** | 单个工具输出过长 | 否 | 清除旧工具输出内容 |
| **Session Memory 压缩** | 自动触发（实验性） | 否 | 用已有记忆替换旧对话 |
| **传统摘要压缩** | 手动 `/compact` 或兜底 | 是 | AI 生成对话摘要 |

**压缩后保留什么**：
- 最近读取的文件（最多 5 个）
- 已激活的技能指令
- CLAUDE.md 内容
- 最近几轮对话

---

### 项目记忆：跨对话持久化

记忆系统存储在文件中，路径：

```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md              ← 入口索引
├── user_role.md           ← 用户角色/偏好
├── feedback_testing.md    ← 用户纠正
├── project_xxx.md         ← 项目上下文
└── reference_xxx.md       ← 外部系统指针
```

**四类型记忆**：

| 类型 | 内容 | 触发场景 |
|-----|------|---------|
| **user** | 用户角色、技术背景 | "我是数据科学家" |
| **feedback** | 用户纠正/确认 | "别 mock 数据库" |
| **project** | 非代码可推导的上下文 | "周四开始合并冻结" |
| **reference** | 外部系统指针 | "bug 在 Linear INGEST 项目" |

**核心约束**：只存储**无法从当前项目状态推导**的信息。代码架构、文件路径、git 历史都可以实时获取，不需要记忆。

---

### 回答第一步的问题：编排机制是什么

第一步我们提到：**"prompt 背后有一整套编排机制"**。具体是什么？

| 编排环节 | 作用 | 说明 |
|---------|------|------|
| **动态组装** | 每次对话重新构建 System Prompt | 根据当前模型能力、MCP 连接状态动态调整 |
| **静态区/动态区划分** | 区分不变内容与变化内容 | 静态区可跨组织缓存，动态区每次会话不同 |
| **Section 注册** | 模块化组装各部分内容 | `system Prompt 数组`，缓存式/非缓存式 Section |
| **CLAUDE.md 注入** | 项目知识自动加载 | 多级合并，自动放入 User Context |
| **记忆召回** | 按相关性筛选记忆 | Sonnet 侧查询，避免无关记忆占 token |

这就是为什么"拿到和 Claude Code 完全相同的 prompt"是不可能的——prompt 是动态组装的，不是一个静态字符串。

---

### 实现细节索引

以下细节对理解核心机制非必需，但有助于深入源码：

| 主题 | 说明 | 入口文件 |
|-----|------|---------|
| **缓存分块策略** | System Prompt 如何分块、标记 `cache_control` | `src/utils/api.ts:321` (`splitSysPromptPrefix`) |
| **System Prompt 组装** | 静态区/动态区划分、Section 注册 | `src/constants/prompts.ts:444` (`getSystemPrompt`) |
| **CLAUDE.md 加载** | 多级合并、目录遍历 | `src/utils/claudemd.ts` (`getClaudeMds`) |
| **Token 计数** | 近似估算 vs 精确计数 | `src/services/tokenEstimation.ts` |
| **自动压缩触发** | 阈值判断、逃逸条件 | `src/services/compact/autoCompact.ts` |
| **MicroCompact** | 工具输出清理、白名单 | `src/services/compact/microCompact.ts` |
| **传统摘要压缩** | 图片剥离、摘要生成、上下文恢复 | `src/services/compact/compact.ts` |
| **Session Memory 压缩** | 保留窗口计算、工具对完整性保护 | `src/services/compact/sessionMemoryCompact.ts` |
| **Compact Boundary** | 压缩边界标记、消息链重建 | `src/utils/messages.ts` (`createCompactBoundaryMessage`) |
| **记忆系统** | 四类型分类、智能召回、注入链路 | `src/memdir/memdir.ts`、`src/memdir/memoryTypes.ts` |
| **记忆召回** | Sonnet 侧查询、相关性筛选 | `src/memdir/findRelevantMemories.ts` |

---

## 第五步：工具系统

**核心问题**：大模型只能输出文本。如何让 AI 真正"动手"——读取文件、修改代码、执行命令？

工具是 AI 的"双手"。AI 发出 `tool_use` 指令，工具系统替它完成实际操作。

---

### getTools() 的分层组装

`src/tools.ts` 的 `getAllBaseTools()` 按四个层级组装工具列表：

```
┌─────────────────────────────────────────────────────────────┐
│ 第一层：固定工具（始终可用）                                  │
│   FileReadTool, FileEditTool, FileWriteTool, BashTool,      │
│   AgentTool, WebFetchTool, WebSearchTool, ...               │
├─────────────────────────────────────────────────────────────┤
│ 第二层：条件工具（运行时检查 isEnabled()）                    │
│   GlobTool, GrepTool, TaskCreateTool, WorktreeTool, ...     │
├─────────────────────────────────────────────────────────────┤
│ 第三层：Feature-flag 工具                                    │
│   feature('KAIROS') ? [SleepTool, ...] : []                 │
├─────────────────────────────────────────────────────────────┤
│ 第四层：MCP 工具（外部服务提供）                              │
│   由连接的 MCP Server 动态注入                               │
└─────────────────────────────────────────────────────────────┘
```

**关键设计**：工具列表在每次 API 调用时组装（而非全局缓存），因为 `isEnabled()` 的结果可能随运行时状态变化。

---

### 第一层：固定工具（核心工具）

这些工具始终可用，构成 AI 的基础能力：

**文件操作**：
| 工具 | 作用 |
|-----|------|
| **FileReadTool** | 读取文件（文本/图片/PDF/Notebook），只读免审批 |
| **FileEditTool** | 精确字符串替换，需先 Read 过该文件 |
| **FileWriteTool** | 全量写入或创建文件 |

**命令执行**：
| 工具 | 作用 |
|-----|------|
| **BashTool** | 执行 shell 命令，只读命令自动放行，有副作用需确认 |

**网络能力**：
| 工具 | 作用 |
|-----|------|
| **WebSearchTool** | 搜索互联网（API 搜索 / Bing 页面解析） |
| **WebFetchTool** | 抓取网页内容，有域名黑名单防护 |

**对话与规划**：
| 工具 | 作用 |
|-----|------|
| **AgentTool** | 派生子 Agent 处理子任务 |
| **AskUserQuestionTool** | 向用户提问获取澄清 |
| **SkillTool** | 执行技能（/斜杠命令） |
| **EnterPlanModeTool** | 进入计划模式（工具集受限为只读） |

---

### 第二层：条件工具

根据运行时状态决定是否启用：

| 条件 | 工具 |
|-----|------|
| 无内嵌搜索工具时 | **GlobTool**（按文件名搜索）、**GrepTool**（按内容搜索） |
| Todo V2 启用时 | **TaskCreate / TaskUpdate / TaskList / TaskGet**（任务管理） |
| Worktree 模式时 | **EnterWorktree / ExitWorktree**（工作树隔离） |
| Agent Swarms 启用时 | **Teams 相关工具**（多 Agent 协作） |
| Windows 平台 | **PowerShellTool** |

---

### 第三层：Feature-flag 工具

通过 feature flag 控制的实验性工具：

| Feature Flag | 工具 |
|-------------|------|
| `KAIROS` | SleepTool, SendUserFileTool, ...（长驻助手模式） |
| `WEB_BROWSER_TOOL` | WebBrowserTool（浏览器控制） |

---

### 第四层：MCP 工具

由 MCP Server 提供的外部工具，通过 `mcpInfo` 字段标记来源：

```
mcp__<server_name>__<tool_name>
```

例如：`mcp__postgres__query`、`mcp__github__create_issue`

MCP 工具支持 blanket deny 规则（按 server 前缀整体禁用）。

---

### 工具调用的统一流程

所有工具都遵循相同的调用链路：

```
tool_use block → validateInput → canUseTool → checkPermissions → call
                      ↓              ↓              ↓              ↓
                   参数校验       权限弹窗        规则匹配       执行操作
```

---

### 回答第一步的问题：治理流水线是什么

第一步我们提到：**"工具背后有一整条治理流水线"**。具体是什么？

| 治理环节 | 作用 | 示例 |
|---------|------|------|
| **分层组装** | 按条件动态加载工具，避免冗余 | `isEnabled()` 运行时检查 |
| **权限分级** | 只读自动放行，写入需确认 | Read 免审批，Edit/Write 需确认 |
| **命令解析** | 拆分复合命令，逐段检查安全性 | `ls && git push` 拆分后分别判定 |
| **输出预算** | 超长结果截断或持久化，防止撑爆上下文 | `maxResultSizeChars` 限制 |
| **并发控制** | 只读工具可并行，有副作用串行执行 | `isConcurrencySafe()` 判定 |
| **超时后台化** | 长命令自动后台化，不阻塞主循环 | 15 秒阈值 |

这就是为什么"拿到和 Claude Code 完全相同的 prompt 和工具集，弄出来的 agent 体验差很多"——你只拿到了工具定义，没拿到这条治理流水线。

---

### 实现细节索引

| 主题 | 入口文件 |
|-----|---------|
| **Tool 类型定义** | `src/Tool.ts` |
| **工具注册** | `src/tools.ts:191` (`getAllBaseTools`) |
| **Read 工具** | `src/tools/FileReadTool/` |
| **Edit 工具** | `src/tools/FileEditTool/` |
| **Write 工具** | `src/tools/FileWriteTool/` |
| **Bash 工具** | `src/tools/BashTool/` |
| **Glob/Grep 工具** | `src/tools/GlobTool/`、`src/tools/GrepTool/` |
| **Agent 工具** | `src/tools/AgentTool/` |
| **Task 工具** | `src/tools/TaskCreateTool/` 等 |
| **WebSearch/WebFetch** | `src/tools/WebSearchTool/`、`src/tools/WebFetchTool/` |
| **MCP 集成** | `src/mcp/` |

---

## 总结：Claude Code 的核心定位

Claude Code 是一个**生产级的 terminal-native agentic coding system**。它的本质是：

> 在大模型的文本生成能力之上，构建了一套完整的 **ReAct 循环**（思考→行动→观察），配合五层架构（交互→编排→循环→工具→通信）、自愈状态机、双重安全模型、分层上下文工程，使 AI 能够**自主地**在真实开发环境中完成软件工程任务。
