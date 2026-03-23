# Toolbox/Event/ToolCallSucceeded.php 分析报告

## 概述

`ToolCallSucceeded` 事件在工具方法成功执行并返回结果后分发，携带工具实例、元数据、调用参数和执行结果，适用于成功审计、结果缓存或日志记录。

## 携带信息

| 字段 | 类型 | 说明 |
|---|---|---|
| `$tool` | `object` | 工具对象实例 |
| `$metadata` | `Tool` | 工具元数据 |
| `$arguments` | `array<string, mixed>` | 调用参数 |
| `$result` | `ToolResult` | 包含工具执行结果和来源的对象 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 工具执行成功后分发此事件 |
| `ToolResult` | 事件携带的执行结果 |
