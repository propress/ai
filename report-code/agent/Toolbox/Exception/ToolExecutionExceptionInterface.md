# Toolbox/Exception/ToolExecutionExceptionInterface.php 分析报告

## 概述

`ToolExecutionExceptionInterface` 继承自 `Toolbox\Exception\ExceptionInterface`，是所有「工具执行失败」类异常的标记接口，强制要求实现 `getToolCallResult(): mixed` 方法以返回可供 LLM 理解的错误描述。

## 关键方法

### getToolCallResult(): mixed

返回工具调用失败时应该传递给 LLM 的结果内容（通常是人类可读的错误字符串），`FaultTolerantToolbox` 捕获此类异常时，调用此方法将错误转为 `ToolResult`：

```php
catch (ToolExecutionExceptionInterface $e) {
    return new ToolResult($toolCall, $e->getToolCallResult());
}
```

## 设计意义

将「工具执行失败」与「工具未找到」、「工具定义错误」区分为不同的异常体系：
- `ToolExecutionExceptionInterface`：运行时执行失败，`FaultTolerantToolbox` 会将其转换为错误消息。
- `ToolNotFoundException`、`ToolException`：配置/定义错误，表示系统级问题。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Exception/ExceptionInterface` | 继承的父接口 |
| `ToolExecutionException` | 内置实现 |
| `FaultTolerantToolbox` | 捕获此接口的异常并调用 `getToolCallResult()` |
| `Toolbox` | 重新抛出此类异常，让 `FaultTolerantToolbox` 处理 |
