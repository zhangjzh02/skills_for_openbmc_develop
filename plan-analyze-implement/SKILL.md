---
name: plan-analyze-implement
description: >
  When the user asks to analyze code and make modifications ("分析xxx", "修改代码",
  "补全功能", "fix this code", "add feature to", "对比旧代码和新代码"), follow
  this structured workflow: explore → plan → review → implement.
  Also triggers on: code comparison tasks, feature migration from old to new,
  gap analysis between two implementations.
---

# 代码分析 → 计划 → 实现 结构化工作流

## 触发条件

当用户请求满足以下任一条件时，应使用此工作流：
- 分析代码并修改（"分析xxx文件，修改xxx"）
- 对比旧代码和新代码，补全功能
- 跨架构移植功能（如 PECI → TPMI）
- 需要探索代码库后制定方案再执行的任何非简单任务

**不适用场景**：单行修复、typo 修改、纯阅读/分析（不修改代码）。

## 工作流总览

```
1. 理解需求（对话中完成，不需要 EnterPlanMode）
2. 进入 Plan Mode（EnterPlanMode）
3. Phase 1: 探索代码库（Explore agent，最多 3 个并行）
4. Phase 2: 设计方案（Plan agent）
5. Phase 3: 审查方案（Read 关键文件 + AskUserQuestion 澄清）
6. Phase 4: 写最终计划到 plan file
7. Phase 5: ExitPlanMode 提交审批
8. 审批后执行（退出 plan mode 后逐步实现）
```

## 详细步骤

### 1. 理解需求

在进入 plan mode 之前：
- 确认用户要分析哪些文件/目录
- 确认用户要实现什么功能
- 如果是对比类任务，确认旧代码和新代码的路径

### 2. 进入 Plan Mode

调用 `EnterPlanMode` 工具。进入后只能读不能写（plan file 除外）。

### 3. Phase 1: 探索代码库

启动 **最多 3 个 Explore agent 并行**：

```json
// Agent 1: 探索目标代码
{
  "description": "Explore target code",
  "prompt": "Read all files in <target-dir>. Focus on: class structure, member variables, methods, existing patterns...",
  "subagent_type": "Explore"
}

// Agent 2: 探索旧/参考代码（如果有）
{
  "description": "Explore reference code",
  "prompt": "Read <reference-file>. Extract: all functions, their signatures, logic flow, external dependencies...",
  "subagent_type": "Explore"
}

// Agent 3: 探索依赖和工具函数
{
  "description": "Explore utilities and dependencies",
  "prompt": "Search for utility functions, D-Bus helpers, logging macros, build configs used by <target>...",
  "subagent_type": "Explore"
}
```

**Agent 数量原则**：
- 只有 1-2 个已知文件 → 1 个 agent
- 多个子系统或新旧对比 → 2-3 个 agent
- 质量优于数量，最多 3 个

### 4. Phase 2: 设计方案

启动 **1 个 Plan agent**：

```
作为 Plan agent，基于以下探索结果设计实现方案：

## 背景
- 目标代码路径：<path>
- 参考代码路径：<path>
- 用户需求：<summary>

## 探索结果
<Phase 1 的关键发现>

## 要求
1. 列出所有需要修改的文件（绝对路径）
2. 每个文件的具体修改内容（新增什么、修改什么）
3. 设计决策和理由
4. 验证方案
```

### 5. Phase 3: 审查方案

- Read Plan agent 提到的最关键文件，确认方案可行
- 如果有歧义，用 `AskUserQuestion` 向用户澄清
- 确保方案与用户原始需求对齐

### 6. Phase 4: 写最终计划

用 `Write` 写入 plan file（路径由系统指定），格式：

```markdown
# Plan: <简短标题>

## Context

<为什么做这个改动 — 要解决的问题、触发原因、预期结果>

## Files to Modify

### N. `<file-name>` — `<absolute-path>`

#### Nx. <子步骤标题>

<具体代码变更描述>

## Key Design Decisions

- **<决策1>**: <理由>
- **<决策2>**: <理由>

## Verification

1. <验证步骤1>
2. <验证步骤2>
```

**计划原则**：
- 用自然语言描述变更，不要逐行贴代码（除非是关键逻辑）
- 引用已有函数/模式时注明文件路径
- 简洁但足够详细，让人能直接执行

### 7. Phase 5: 提交审批

调用 `ExitPlanMode`。不要用 AskUserQuestion 问"方案可以吗"——ExitPlanMode 就是审批机制。

### 8. 执行实现

审批通过后：
1. 创建 task list（TaskCreate）跟踪进度
2. 逐步实现，每完成一个 task 标记 completed
3. 修改完成后做最终验证（检查修改点是否齐全）

## 反模式（避免）

- ❌ 跳过 Explore agent，直接自己读文件 → Plan agent 缺少完整上下文
- ❌ 跳过 Plan agent，自己设计方案 → 容易遗漏边界情况
- ❌ 在 plan mode 中就开始编辑代码 → plan mode 只能读和写 plan file
- ❌ 计划写得太简略（只有文件名没有修改内容）→ 无法有效审批
- ❌ 计划写得太详细（逐行代码）→ 冗余，关键决策反而被淹没
- ❌ 用 AskUserQuestion 问"方案可以吗" → 应该用 ExitPlanMode

## 适用判断速查

| 任务特征 | 是否用此工作流 |
|----------|---------------|
| 修改涉及 2+ 个文件 | ✅ 用 |
| 需要理解现有逻辑才能修改 | ✅ 用 |
| 新旧代码对比迁移 | ✅ 用 |
| 单文件单行 fix | ❌ 直接改 |
| typo/注释修改 | ❌ 直接改 |
| 纯阅读分析不修改 | ❌ 直接用 Explore agent |
