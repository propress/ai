# Toolbox/ToolboxInterface.php 分析报告

## 概述

`ToolboxInterface` 定义了工具箱的公开契约，包含获取工具元数据列表和执行单次工具调用两个方法，是 `AgentProcessor` 与 Toolbox 实现之间的解耦点。

## 关键方法分析

### getTools(): Tool[]

- 返回所有已注册工具的元数据（`Platform\Tool\Tool` 对象，含名称、描述、参数 Schema）。
- `AgentProcessor::processInput()` 调用此方法，将工具列表注入 `options['tools']`，Platform 模块再将其转换为 API 请求中的 `tools` 参数。

### execute(ToolCall $toolCall): ToolResult

- 执行单次工具调用，返回 `ToolResult`（含原始调用对象和执行结果）。
- 声明抛出 `ToolExecutionExceptionInterface`（执行失败）和 `ToolNotFoundException`（工具不存在）。
- `FaultTolerantToolbox` 通过实现此接口并捕获上述异常，将错误转换为 LLM 可理解的错误消息。

## 内置实现

| 实现类 | 特点 |
|---|---|
| `Toolbox` | 标准实现，含事件分发和日志 |
| `FaultTolerantToolbox` | 装饰器，捕获异常并返回错误消息字符串 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 主要实现 |
| `FaultTolerantToolbox` | 容错包装实现 |
| `AgentProcessor` | 持有 `ToolboxInterface`，调用 `getTools()` 和 `execute()` |
| `SystemPromptInputProcessor` | 可选依赖此接口，将工具描述注入系统提示 |
| `ToolResult` | `execute()` 返回类型 |
