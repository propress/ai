# Toolbox/FaultTolerantToolbox.php 分析报告

## 概述

`FaultTolerantToolbox` 是 `ToolboxInterface` 的装饰器实现，包装内部工具箱实例，捕获工具执行异常并将其转换为描述性错误字符串，使 LLM 能够理解工具失败原因并决定后续行动，而不是让异常冒泡中断整个 Agent 调用。

## 关键方法分析

### execute(ToolCall $toolCall): ToolResult

```php
try {
    return $this->innerToolbox->execute($toolCall);
} catch (ToolExecutionExceptionInterface $e) {
    return new ToolResult($toolCall, $e->getToolCallResult());
} catch (ToolNotFoundException) {
    $names = array_map(fn (Tool $m) => $m->getName(), $this->getTools());
    return new ToolResult($toolCall, sprintf('Tool "%s" was not found, please use one of these: %s', ...));
}
```

- **`ToolExecutionExceptionInterface`**：调用 `getToolCallResult()` 获取人类可读的错误消息字符串，包装为 `ToolResult` 返回。
- **`ToolNotFoundException`**：生成包含所有可用工具名称的提示消息，帮助 LLM 纠正工具名称后重试。

### getTools(): Tool[]

直接委托给 `$innerToolbox->getTools()`，不添加任何逻辑。

## 设计模式

- **装饰器模式（Decorator）**：包装 `ToolboxInterface` 实例，增加容错行为而不修改原实现。
- **容错机制（Fault Tolerance）**：将异常转换为数据（错误字符串），维持系统的正常运行流程，让 LLM 有机会自我纠错。

## 使用建议

在需要健壮性的生产环境中，推荐将 `Toolbox` 包装在 `FaultTolerantToolbox` 中。如果需要让工具执行失败中断整个 Agent 调用，则直接使用 `Toolbox`。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolboxInterface` | 实现此接口，同时持有 `ToolboxInterface` 实例（装饰器） |
| `Toolbox` | 通常被此类包装 |
| `ToolResult` | 包装错误信息时创建的对象 |
| `Exception/ToolExecutionExceptionInterface` | 捕获此类异常并转化为错误字符串 |
| `Exception/ToolNotFoundException` | 捕获此类异常并提供可用工具列表 |
