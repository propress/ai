# CompletionsResultConverter.php 分析报告

## 概述

`CompletionsResultConverter` 继承自 `Generic\Completions\ResultConverter`，专门修复 amazee.ai 使用的 LiteLLM 代理在处理结构化输出时的特殊行为：`finish_reason` 为 `tool_calls` 但响应内容位于 `message.content` 而非 `message.tool_calls` 中。

## 关键方法分析

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，明确禁用 Token 用量统计，因为 LiteLLM 代理层的 Token 统计数据不可靠。

### `convertChoice(array $choice): ToolCallResult|TextResult`
重写父类的选择转换逻辑，处理以下三种场景：

1. **正常工具调用**：`finish_reason === 'tool_calls'` 且 `message.tool_calls` 存在 → 返回 `ToolCallResult`
2. **LiteLLM 结构化输出兼容**：`finish_reason === 'tool_calls'` 但 `message.content` 存在 → 返回 `TextResult`（LiteLLM quirk 修复）
3. **正常文本/长度截断**：`finish_reason` 为 `stop` 或 `length` → 返回 `TextResult`

不匹配以上任何情况时，抛出 `RuntimeException`。

## 设计模式

- **模板方法模式（Template Method）**：父类 `ResultConverter` 定义转换流程框架，子类通过重写 `convertChoice()` 插入特定的兼容性逻辑
- **防御性编程**：优先检查具体 `tool_calls` 字段，再回退到 `content` 字段，确保不遗漏任何有效数据

## 与其他类的关系

- 替代 `Generic\Completions\ResultConverter` 在 `PlatformFactory` 中使用
- 实现 `ResultConverterInterface` 契约，由 `Platform` 核心在响应转换阶段调用
