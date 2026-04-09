# 零基础 学CC

根据文档中 **[为什么写这份白皮书](/docs/introduction/why-this-whitepaper)** 页面提供的官方阅读路线图，推荐学习路径如下：

**🗺️ 推荐学习路径**

1. **建立直觉** → Introduction（什么是 Claude Code → 架构全景）
2. **安全体系** → 权限模型 → 沙箱机制 → Plan Mode
3. **对话引擎** → Agentic Loop → 流式响应 → 多轮对话
4. **上下文工程** → System Prompt → Token 预算 → 项目记忆
5. **工具系统** → 工具概览 → Shell 执行 → 搜索与导航
6. **Agent 与扩展** → 子 Agent → 自定义 Agent → MCP 协议
7. **进阶** → Features（实验功能）→ Internals（内部机制如 Feature Flags、A/B 测试）

核心思路是：**先理解全貌，再深入安全边界，然后学习核心循环和上下文管理，最后探索工具和扩展能力**。


## 大模型是什么

大语言模型（LLM）核心能力是**理解和生成自然语言**。它接收一段文本（prompt），输出一段续写文本（completion）。为了方便理解，我们可以认为它是一个**无状态的文本生成函数**。

关键局限：
- **无状态**：不记得上一次对话
- **无行动力**：只能输出文本，不能读文件、跑命令
- **上下文有限**：输入 token 有上限

---

## Agent 是什么

Agent 是在大模型之上增加了**感知-决策-行动**循环的系统。最经典的范式是 **ReAct（Reasoning + Acting）**：

```
用户输入 → 思考(Reasoning) → 行动(Acting) → 观察(Observation) → 再思考 → ... → 输出答案
```

| ReAct 步骤 | 含义 |
|------------|------|
| **Reasoning（思考）** | 模型分析当前状态，决定下一步做什么 |
| **Acting（行动）** | 调用外部工具（API、文件操作、命令执行等） |
| **Observation（观察）** | 获取工具返回的真实结果，作为下一轮思考的输入 |

与纯聊天机器人的区别：**有工具**（能调用外部 API、读写文件、执行命令）、**有循环**（多步推理）、**有决策权**（AI 自己决定用哪个工具、传什么参数、何时停止）。


基础的 ReAct(思考→行动→观察)解决了 LLM 无状态无行动力上下文隔离的核心问题，现在假设我们拿到了和 Claude Code 完全相同的 system prompt 和工具集，我们这个简单的 agent 能够像 cc 一样优秀的工作吗？很显然是不能的，因为我们很快会遇到这些问题：API 报错直接崩溃、对话太长 token 超限直接崩溃、AI 执行了危险命令如`rm -rf /`把我们库给删除了、需要同时改 5 个文件只能一个一个串行删除等等问题。

和 Claude Code 一样的 prompt、一样的工具弄出来的 agent，最终体验效果和 cc非常大？差距在哪？差在 prompt 背后有一整套编排机制，工具背后有一整条治理流水线，Agent 背后有一套分工和调度系统，生态扩展背后有一套让模型"知道自己能干什么"的感知通道。这些东西加在一起，才构成了你用 Claude Code 时感受到的那个"稳"。这个稳不是来自某段神奇文案，是来自工程。

---

## Claude Code 是什么

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

## Claude Code 的架构与设计

### 五层架构

| 层次 | 职责 | 入口源码 | 关键词 |
|------|------|----------|--------|
| **交互层** | 终端 UI、用户输入、消息展示 | `src/screens/REPL.tsx` | React/Ink、PromptInput |
| **编排层** | 多轮对话、会话持久化、成本追踪 | `src/QueryEngine.ts` | QueryEngine、transcript |
| **核心循环层** | 发请求 → 拿响应 → 执行工具 → 循环 | `src/query.ts` | Agentic Loop、State |
| **工具层** | AI 的"双手"——读写文件、执行命令（50+ 工具） | `src/tools.ts` → `src/Tool.ts` | Tool 接口、MCP |
| **通信层** | 与 Claude API 的流式通信 | `src/services/api/claude.ts` | Streaming、Provider |

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

#### QueryEngine：会话编排器

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

#### 会话持久化

```
~/.claude/projects/<project-hash>/<session-id>.jsonl
```

JSONL 格式支持追加写入，`--resume` 从历史消息恢复会话。包含成本追踪（`totalCostUSD`、按模型分拆用量）和文件快照（绑定到 message.id，支持 `--rewind-files` 精确回滚）。

## 总结：Claude Code 的核心定位

Claude Code 是一个**生产级的 terminal-native agentic coding system**。它的本质是：

> 在大模型的文本生成能力之上，构建了一套完整的 **ReAct 循环**（思考→行动→观察），配合五层架构（交互→编排→循环→工具→通信）、自愈状态机、双重安全模型、分层上下文工程，使 AI 能够**自主地**在真实开发环境中完成软件工程任务。
