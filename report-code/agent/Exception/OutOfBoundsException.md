# Exception/OutOfBoundsException.php 分析报告

## 概述

`OutOfBoundsException` 是 Agent 模块中表示「索引越界」的异常类，继承自 PHP 内置的 `\OutOfBoundsException` 并实现 `ExceptionInterface`，用于访问超出有效范围的集合元素时抛出。

## 抛出场景

- `MockAgent::getCall(int $index)`：当请求的调用记录索引不存在时抛出，错误消息包含请求的索引和实际调用次数。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Exception/ExceptionInterface` | 实现此接口 |
| `MockAgent` | 在 `getCall()` 索引越界时抛出 |
