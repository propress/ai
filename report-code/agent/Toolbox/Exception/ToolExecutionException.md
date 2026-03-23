# Toolbox/Exception/ToolExecutionException.php 分析报告

## 概述

`ToolExecutionException` 是 `ToolExecutionExceptionInterface` 的标准实现，在工具方法抛出非预期异常时由 `Toolbox` 包装并抛出，提供人类可读的错误消息供 `FaultTolerantToolbox` 转交给 LLM。

## 关键方法分析

### executionFailed(ToolCall $toolCall, Throwable $previous): self（静态构造器）

包装原始异常，生成错误消息：
`'Execution of tool "..." failed with error: {原始错误消息}'`

同时将 `$toolCall` 存储到实例中，以便 `getToolCall()` 后续访问。

### getToolCallResult(): string

实现 `ToolExecutionExceptionInterface` 要求的方法，返回 LLM 友好的错误描述：
`'An error occurred while executing tool "...".'`

注意：此消息有意模糊（不暴露内部错误细节），避免向 LLM 泄露实现信息。

### getToolCall(): ?ToolCall

返回关联的工具调用请求，可用于调试。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolExecutionExceptionInterface` | 实现此接口 |
| `Toolbox` | 在捕获非 `ToolExecutionExceptionInterface` 的 `\Throwable` 时创建此异常并抛出 |
| `FaultTolerantToolbox` | 捕获此类异常，调用 `getToolCallResult()` 获取 LLM 友好错误消息 |
