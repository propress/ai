# Exception/InvalidArgumentException.php 分析报告

## 概述

`InvalidArgumentException` 是 Agent 模块中表示「无效参数」的异常类，继承自 PHP 内置的 `\InvalidArgumentException` 并实现 `ExceptionInterface`，使其既符合 PHP 标准异常语义，又可被模块级 catch 语句捕获。

## 抛出场景

- `Agent::call()`：处理器未实现正确接口时。
- `Agent::call()`：`options['model']` 非字符串时（由 `ModelOverrideInputProcessor` 触发）。
- `MultiAgent` 构造函数：`$handoffs` 数组为空时。
- `Handoff` 构造函数：`$when` 条件数组为空时。
- `Toolbox/Exception/ToolConfigurationException`：继承自本类。
- `Toolbox/Exception/ToolException`：继承自本类。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Exception/ExceptionInterface` | 实现此接口 |
| `Toolbox/Exception/ToolConfigurationException` | 继承本类 |
| `Toolbox/Exception/ToolException` | 继承本类 |
