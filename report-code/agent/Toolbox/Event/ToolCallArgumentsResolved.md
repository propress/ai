# Toolbox/Event/ToolCallArgumentsResolved.php 分析报告

## 概述

`ToolCallArgumentsResolved` 事件在工具参数完成反序列化、即将调用工具方法之前分发，携带已解析为强类型 PHP 值的参数数组，适用于参数审计或日志记录。

## 携带信息

| 字段 | 类型 | 说明 |
|---|---|---|
| `$tool` | `object` | 工具对象实例 |
| `$metadata` | `Tool` | 工具元数据（含方法名、描述、参数 Schema）|
| `$arguments` | `array<string, mixed>` | 已反序列化的参数（可直接用于方法调用）|

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 在参数解析完成后、方法调用前分发此事件 |
| `ToolCallArgumentResolver` | 生成 `$arguments` 的组件 |
