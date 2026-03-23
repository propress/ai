# Exception/ExceptionInterface.php 分析报告

## 概述

`ExceptionInterface` 是 Agent 模块所有异常类的根标记接口，继承自 PHP 内置的 `\Throwable`，用于通过 `catch (ExceptionInterface $e)` 捕获模块内所有受控异常，同时不破坏标准 PHP 异常层级。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InvalidArgumentException` | 实现此接口 |
| `LogicException` | 实现此接口 |
| `RuntimeException` | 实现此接口 |
| `OutOfBoundsException` | 实现此接口 |
| `MaxIterationsExceededException` | 间接实现（继承 `RuntimeException`） |
| `Toolbox/Exception/ExceptionInterface` | 继承此接口，扩展 Toolbox 子模块异常体系 |
