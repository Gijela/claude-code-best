# Prompt 编排机制

> **核心问题**：AI 怎么理解上下文？
>
> **答案**：不是靠一段写死的 prompt，而是靠一套**动态组装、预算管理、自动压缩**的编排机制。

---

## 概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Prompt 编排机制全景                        │
├─────────────────────────────────────────────────────────────┤
│  1. CLAUDE.md         ← 项目级知识注入（最低成本定制）         │
│  2. 环境信息注入       ← Git 状态、日期、模型信息              │
│  3. System Prompt     ← 动态组装 + 缓存分块                   │
│  4. Token 预算        ← 200K 不是全部，预留管理               │
│  5. 四层压缩          ← 时间/缓存编辑/记忆/摘要递进            │
│  6. 项目记忆          ← 跨对话持久化                          │
└─────────────────────────────────────────────────────────────┘
```

---

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

### Feature Flag 对 System Prompt 的影响

| Feature Flag | 影响 |
|-------------|------|
| `PROACTIVE` / `KAIROS` | 完全切换为自主代理模式，System Prompt 大幅简化为 `You are an autonomous agent...` |
| `EXPERIMENTAL_SKILL_SEARCH` | 增加 DiscoverSkills 工具使用指南 |
| `CACHED_MICROCOMPACT` | 增加 Function Result Clearing Section |
| `TOKEN_BUDGET` | 增加 token 预算目标指导 |
| `CONTEXT_COLLAPSE` | 抑制主动压缩，交由 Context Collapse 机制管理 |

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

---

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

---

## 核心价值总结

| 组件 | 解决的问题 | 为什么重要 |
|------|-----------|-----------|
| **CLAUDE.md** | 项目上下文注入 | 最低成本定制 |
| **环境信息注入** | AI 感知当前状态 | 知道"在哪"、"用什么" |
| **动态组装** | 缓存友好 | 节省 token 成本 |
| **Token 预算** | 上下文上限管理 | 防止崩溃 |
| **四层压缩** | 自动恢复 | 用户无感知 |
| **项目记忆** | 跨对话连续性 | 不用重复说明 |
