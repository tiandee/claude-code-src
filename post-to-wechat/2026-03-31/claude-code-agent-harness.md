---
title: 从 Claude Code 源码学习 Agent Harness 架构设计
author: TianDee
description: 深入分析 Anthropic 开源的 Claude Code 核心源码，提取 7 大生产级 Agent 框架设计模式，为构建自己的 Agent Harness 提供架构蓝本。
---

# 从 Claude Code 源码学习 Agent Harness 架构设计

最近我拿到了一份 Claude Code 的核心源码（内部代号 Tengu），花了不少时间做逆向分析。作为一个正在设计 Agent Harness 框架的开发者，这份代码给了我非常多启发。本文提炼出其中最值得借鉴的 **7 个核心设计模式**，希望对同样在做 Agent 框架的同学有用。

## 一、AsyncGenerator 驱动的流式主循环

这是整个 Claude Code 最优雅的架构选择。

核心文件 `query.ts`（68KB）用 `async function*` 实现了一个无限循环的查询引擎：

```typescript
async function* queryLoop(state: QueryState): AsyncGenerator<StreamEvent> {
  while (true) {
    // 1. 上下文压缩检查
    // 2. 调用 LLM API（流式）
    // 3. 流式执行工具
    // 4. 执行 Stop Hooks
    // 5. 判断是否继续循环
    yield* streamEvents;
  }
}
```

为什么用 AsyncGenerator 而不是普通的 async/await 循环？

- **天然支持流式输出**：每有一个 token 或工具进度，立即 `yield` 给上层渲染
- **背压控制**：消费者（UI 层）处理不过来时，生产者自动暂停
- **状态保持**：循环间的 State 对象天然在闭包中保持，不需要额外的状态管理
- **可组合**：通过 `yield*` 委托子生成器，实现了工具执行、压缩等子流程的无缝嵌套

> **设计启示**：如果你的 Agent Harness 需要流式输出能力，AsyncGenerator 是比回调或 EventEmitter 更优雅的选择。

## 二、边流式边执行的工具编排

传统的 Agent 框架是"等 LLM 返回完整响应 → 解析 tool_use → 执行工具"。Claude Code 不是——它**边接收 LLM 的流式输出，边启动工具执行**。

核心是 `StreamingToolExecutor`，它维护了一个状态机：

```
queued → executing → completed → yielded
```

当 LLM 流式输出第一个 `tool_use` 块时，Executor 立即检查该工具是否可以执行（不依赖后续输出），如果可以就立即启动——不等 LLM 把整条消息输出完。

**并发控制**是另一个亮点。每个工具通过 `isConcurrencySafe(input)` 动态声明自己是否可以并行：

- `FileReadTool`、`GrepTool`、`GlobTool` → 始终并发安全
- `FileEditTool` → 始终不安全（需串行）
- `BashTool` → 根据命令内容动态判断（只读命令如 `ls`、`cat` 返回 true）

`toolOrchestration.ts` 中的 `partitionToolCalls()` 函数将一批工具调用划分为：

```
[ConcurrentBatch(grep, cat, ls), SerialBatch(edit), ConcurrentBatch(grep)]
```

同一批次内并行执行，批次间串行。

> **设计启示**：工具的并发安全性不应该是静态标记，而应该根据输入动态判定。一个 `bash` 工具可以并行执行 `cat`，但必须串行执行 `rm`。

## 三、四级递进式上下文压缩

长对话的上下文管理是 Agent 框架最头疼的问题。Claude Code 设计了四级压缩策略，从轻到重递进：

| 级别 | 策略 | API 消耗 | 触发时机 |
|------|------|---------|---------|
| L1 | **Snip Compact** | 零 | 裁剪最早的历史消息 |
| L2 | **Micro Compact** | 零 | 压缩工具执行结果（保留 tool_use_id） |
| L3 | **Context Collapse** | 零 | 折叠已完成的工具调用链为摘要 |
| L4 | **Auto Compact** | 高 | 调用 LLM 生成全文对话总结 |

核心思想是**先尝试不消耗 LLM token 的策略**。只有前三级都无法将上下文降到阈值以下时，才触发昂贵的 L4 自动压缩。

L4 还有一个精巧的**熔断器设计**：如果连续 3 次 Auto Compact 失败（通常意味着对话结构异常），停止重试，避免恶性循环消耗 token。

压缩触发阈值计算也很有意思：

```
threshold = effectiveContextWindow - 13K_buffer
```

预留 13K 是为了给下一轮的系统提示词 + 工具定义 + 新消息留出空间。

> **设计启示**：上下文管理不是一个单一的压缩函数，而是一个多级策略链。优先用"免费"的策略，把 LLM 压缩作为最后手段。

## 四、三种 Multi-Agent 协调模型

Claude Code 内置了三种子 Agent 生命周期模型：

**Fork Subagent**：继承父 Agent 的完整对话上下文。核心创新在于 **Prompt Cache 共享**——通过 `buildForkedMessages()` 构建 byte-identical 的 API 请求前缀，使得所有 Fork 子 Agent 共享同一份 prompt cache，极大降低了成本。

**Worker Subagent**：全新的上下文和系统提示词，独立执行。Agent 定义通过 `.claude/agents/` 目录声明式配置。Worker 有独立的 AbortController 和 FileStateCache。

**Coordinator Mode**：纯编排角色，不直接使用文件工具。通过 4 阶段流程（Research → Synthesis → Implementation → Verification）调度多个 Worker。Worker 结果通过结构化的 XML `<task-notification>` 返回给 Coordinator。

> **设计启示**：不同的任务需要不同的 Agent 模式。简单的并行任务用 Fork（共享上下文），独立的子任务用 Worker（隔离执行），复杂的多步骤任务用 Coordinator（纯编排）。

## 五、分层权限系统

Claude Code 定义了 6 种 `PermissionMode`：

```
default → plan → acceptEdits → auto → bypassPermissions → bubble
```

其中 `bubble` 模式是专门为子 Agent 设计的——子 Agent 的权限请求"冒泡"到父进程，由人类在父进程中审批。

权限规则有三层来源，按优先级排列：

```
CLI 参数 > 会话级设置 > 全局 settings.json
```

Hook 系统可以注入 `blocking error`，这会强制 Agent 继续执行（而不是停止），用于权限校验失败后的修复流程。

## 六、Feature Flag 门控系统

Claude Code 大量使用两种特性门控：

- **编译时门控**：`feature('COORDINATOR_MODE')` 基于 Bun 的 `bun:bundle` 实现，在构建时做 Dead Code Elimination
- **运行时门控**：GrowthBook 远程配置，前缀统一为 `tengu_*`

这让整个系统可以安全地做 A/B 测试和灰度发布，同时对外发布版本可以完全移除内部实验代码。

## 七、精细化成本追踪

按模型、按 Agent 分层追踪 token 消耗：

- 分开统计 `input_tokens` / `output_tokens` / `cache_read` / `cache_creation`
- 每次 compact 单独记录成本
- 成本数据持久化到 session 配置，支持会话恢复时的成本累计

## 总结：可迁移的架构蓝本

从 Claude Code 源码中，我提炼出构建生产级 Agent Harness 最关键的几个设计决策：

1. **主循环用 AsyncGenerator**，而不是简单的 while 循环
2. **工具并发安全性动态判定**，不是静态标记
3. **上下文压缩做多级策略链**，LLM 压缩是最后手段
4. **子 Agent 模式按需选择**，共享上下文 vs 隔离执行
5. **权限模型支持冒泡**，子进程权限可上浮到父进程

这些不是理论上的最佳实践，而是一个日活百万的 Agent 产品经过无数次迭代验证过的工程方案。希望这篇文章能帮你在设计自己的 Agent 框架时少走弯路。

---

> 本文基于对 Claude Code 源码（~3.2MB TypeScript）的深度分析。文中的模块名、文件名、函数名均来自真实代码。
