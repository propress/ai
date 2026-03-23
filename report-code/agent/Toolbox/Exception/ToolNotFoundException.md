# Toolbox/Exception/ToolNotFoundException.php 分析报告

## 概述

`ToolNotFoundException` 在 LLM 请求调用一个 Toolbox 中不存在的工具时抛出，继承自 `\RuntimeException` 并实现 `Toolbox\Exception\ExceptionInterface`，提供两个命名静态构造器。

## 静态构造器

### notFoundForToolCall(ToolCall $toolCall): self

LLM 发出工具调用但工具名称在 Toolbox 中未注册时使用：
`'Tool not found for call: {toolCallName}.'`

### notFoundForReference(ExecutionReference $reference): self

通过类名和方法名查找可执行对象时找不到时使用（`Toolbox::getExecutable()` 中的后备查找）：
`'Tool not found for reference: {ClassName}::{method}.'`

## 处理方式

- `Toolbox::execute()`：直接重新抛出，不做处理。
- `FaultTolerantToolbox::execute()`：捕获后返回包含所有可用工具名称列表的 `ToolResult`，引导 LLM 使用正确工具名称重试。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Exception/ExceptionInterface` | 实现此接口 |
| `Toolbox` | 在 `getMetadata()` 和 `getExecutable()` 中抛出 |
| `FaultTolerantToolbox` | 捕获并转换为包含工具列表的错误 `ToolResult` |
