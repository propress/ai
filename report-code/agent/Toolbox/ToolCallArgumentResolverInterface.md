# Toolbox/ToolCallArgumentResolverInterface.php 分析报告

## 概述

`ToolCallArgumentResolverInterface` 定义了工具调用参数解析器的契约：给定工具元数据和工具调用对象，返回可直接用于方法调用的参数数组。

## 关键方法

### resolveArguments(Tool $metadata, ToolCall $toolCall): array<string, mixed>

- `$metadata`：工具的元数据对象，含方法引用（类名 + 方法名）用于反射。
- `$toolCall`：LLM 生成的工具调用，含 JSON 参数 `array<string, mixed>`。
- 返回解析后的参数数组，可直接用于 `$tool->{method}(...$arguments)` 调用（PHP 命名参数展开）。
- 声明抛出 `ToolException`（参数无法解析时）。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolCallArgumentResolver` | 内置默认实现 |
| `Toolbox` | 构造注入此接口，在 `execute()` 中调用 |
| `Exception/ToolException` | 参数解析失败时抛出 |
