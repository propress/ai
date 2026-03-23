# Toolbox/Event/ToolCallRequested.php 分析报告

## 概述

`ToolCallRequested` 是工具调用生命周期的第一个事件，在工具实际执行之前分发，实现了 `StoppableEventInterface`，允许监听者拒绝（`deny()`）执行或直接提供结果（`setResult()`）以跳过实际工具调用。

## 关键方法分析

### deny(?string $reason = null): void

拒绝工具执行，可附带原因字符串。`Toolbox::execute()` 检测到 `isDenied()` 为 `true` 时，返回含拒绝原因的 `ToolResult`，不调用工具。

### setResult(ToolResult $result): void

直接设置工具结果以跳过实际执行，可用于实现工具调用缓存或权限门控。

### isPropagationStopped(): bool

若已拒绝或已设置结果，停止事件传播。

## 使用场景

- **访问控制**：监听此事件，根据工具名称或参数决定是否允许调用（如限制危险操作）。
- **审计日志**：记录所有工具调用请求。
- **工具缓存**：在结果已缓存时通过 `setResult()` 直接返回，避免重复调用。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 在执行工具前分发此事件，检查 `isDenied()` 和 `hasResult()` |
| `ToolResult` | `setResult()` 接受的类型 |
| `Platform/Result/ToolCall` | 事件携带的工具调用请求 |
