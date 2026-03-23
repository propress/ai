# Toolbox/ToolFactory/ChainFactory.php 分析报告

## 概述

`ChainFactory` 是 `ToolFactoryInterface` 的链式组合实现，依次尝试多个子工厂，返回第一个成功的结果；若所有子工厂均失败，则抛出 `ToolException`，实现工厂优先级链。

## 关键方法分析

### getTool(object|string $reference): iterable<Tool>

```php
foreach ($this->factories as $factory) {
    try {
        yield from $factory->getTool($reference);
    } catch (ToolException) {
        ++$invalid;
        continue;
    }
    return;  // 第一个成功就停止
}
if ($invalid === count($this->factories)) {
    throw ToolException::invalidReference($reference);
}
```

- 若某工厂成功返回（不抛异常），立即结束循环（`return`），不再尝试后续工厂。
- 若某工厂抛出 `ToolException`，记录失败计数并继续下一个工厂。
- 若所有工厂均失败，抛出 `ToolException`。

## 使用场景

典型配置：将 `ReflectionToolFactory`（处理带注解的类）与 `MemoryToolFactory`（处理编程式注册的类）组合，以及将来可能加入的 MCP 或其他来源工厂：

```php
new ChainFactory([
    new ReflectionToolFactory(),
    new MemoryToolFactory(),
    new McpToolFactory($client),
])
```

## 设计模式

- **责任链模式（Chain of Responsibility）**：工厂链，每个工厂尝试处理请求，失败则传递给下一个。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolFactoryInterface` | 实现此接口，同时聚合多个 `ToolFactoryInterface` |
| `ToolFactory/ReflectionToolFactory` | 通常作为链中第一个工厂 |
| `ToolFactory/MemoryToolFactory` | 通常作为链中备选工厂 |
| `Exception/ToolException` | 所有子工厂失败时抛出 |
