# OutputProcessorInterface.php 分析报告

## 概述

`OutputProcessorInterface` 定义了在 Agent 收到 AI 平台响应之后对 `Output` 对象进行变换的扩展点，是 Agent 模块输出管道的核心契约。

## 关键方法分析

### processOutput(Output $output): void

- 接收 `Output` 对象，可读取平台返回的 `ResultInterface` 并通过 `setResult()` 替换为处理后的结果。
- 返回 `void`，通过修改 `$output` 对象的 `result` 字段传递变更。

## 内置实现

| 实现类 | 功能 |
|---|---|
| `AgentProcessor` | 检测 `ToolCallResult`，执行工具调用迭代循环，最终将文本结果写回 `Output` |

## 核心使用场景

`AgentProcessor` 是目前最重要的 OutputProcessor 实现。它在 `processOutput()` 中：
1. 检查 `result` 是否为 `ToolCallResult`（需要执行工具）或 `StreamResult`（流式响应）。
2. 对于工具调用：启动迭代循环，执行所有工具，重新调用 Agent，直到收到最终文本。
3. 对于流式响应：注册 `StreamListener` 以在流中拦截工具调用块。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent` | 遍历并调用处理器链 |
| `Output` | 处理器操作的上下文对象 |
| `Attribute/AsOutputProcessor` | Symfony DI 自动注册注解 |
| `Toolbox/AgentProcessor` | 唯一内置实现，驱动工具调用迭代循环 |
