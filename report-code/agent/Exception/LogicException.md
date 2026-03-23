# Exception/LogicException.php 分析报告

## 概述

`LogicException` 是 Agent 模块中表示「程序逻辑错误」（通常是错误的调用顺序或前置条件不满足）的异常类，继承自 PHP 内置的 `\LogicException` 并实现 `ExceptionInterface`。

## 抛出场景

- `MockAgent::getLastCall()`：在没有任何调用记录时调用此方法，属于调用顺序错误。
- `Bridge/YoutubeTranscriber` 构造函数：`mrmysql/youtube-transcript` 包未安装时（通过继承的 `LogicException` 语义表达「缺少依赖」）。

## 设计理念

`LogicException` 与 `RuntimeException` 的语义区别：
- `LogicException`：表示代码逻辑错误，通常在开发阶段即可发现，应通过修改代码来修复。
- `RuntimeException`：表示运行时环境错误，如网络故障、API 错误等，无法在编译阶段预测。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Exception/ExceptionInterface` | 实现此接口 |
| `MockAgent` | 在 `getLastCall()` 无调用记录时抛出 |
