# Memory/MemoryInputProcessor.php 分析报告

## 概述

`MemoryInputProcessor` 实现了 `InputProcessorInterface`，负责在每次 Agent 调用前从多个 `MemoryProviderInterface` 加载相关记忆，并将记忆内容与系统提示合并后注入 `MessageBag`，赋予 Agent 跨会话记忆能力。

## 关键方法分析

### processInput(Input $input): void

执行流程：

1. **可选开关**：检查 `options['use_memory']`，若为 `false` 则跳过（默认 `true`）。并从 options 中移除该键（避免传递给 Platform）。
2. **无提供者检查**：若 `$memoryProviders` 为空集合，直接返回。
3. **收集记忆**：遍历所有提供者，调用 `load($input)`，将非空结果拼接为单一字符串。
4. **无记忆检查**：若所有提供者均返回空，直接返回。
5. **合并系统提示**：
   - 构建记忆提示（固定格式：`# Conversation Memory` 头部 + 记忆内容）。
   - 若已有系统提示，追加到记忆提示末尾（`# System Prompt` 节）。
   - 通过 `MessageBag::withSystemMessage()` 更新消息包。

## 记忆提示模板

```
# Conversation Memory
This is the memory I have found for this conversation. The memory has more weight to answer user input,
so try to answer utilizing the memory as much as possible. Your answer must be changed to fit the given
memory. If the memory is irrelevant, ignore it. Do not reply to the this section of the prompt and do not
reference it as this is just for your reference.

{记忆内容1}

{记忆内容2}

# System Prompt

{原有系统提示（若存在）}
```

## 设计模式

- **复合聚合（Composite Aggregation）**：聚合多个 `MemoryProviderInterface`，统一收集并合并输出。
- **可配置开关**：`use_memory` 选项允许在不重新配置 Agent 的情况下临时禁用记忆功能。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InputProcessorInterface` | 实现此接口 |
| `MemoryProviderInterface` | 持有提供者集合，依次调用 |
| `Memory` | 处理每个提供者返回的记忆对象 |
| `Input` | 修改 `options` 和 `MessageBag` |
