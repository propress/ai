# Toolbox/Event/ToolCallFailed.php 分析报告

## 概述

`ToolCallFailed` 事件在工具方法抛出异常时分发（在异常被重新抛出之前），携带工具实例、元数据、调用参数和异常对象，适用于错误监控、告警或自定义错误处理。

## 携带信息

| 字段 | 类型 | 说明 |
|---|---|---|
| `$tool` | `object` | 工具对象实例 |
| `$metadata` | `Tool` | 工具元数据 |
| `$arguments` | `array<string, mixed>` | 调用参数（若参数解析失败则为空数组）|
| `$exception` | `\Throwable` | 捕获的异常 |

## 注意事项

此事件仅用于通知，不能阻止异常传播（无 `stopException()` 机制）。异常在分发事件后仍然会被重新抛出（或包装为 `ToolExecutionException` 后抛出），由 `FaultTolerantToolbox` 捕获处理。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 工具执行失败时分发此事件 |
| `FaultTolerantToolbox` | 捕获最终抛出的异常并转换为错误消息 |
