# Toolbox/Toolbox.php 分析报告

## 概述

`Toolbox` 是 `ToolboxInterface` 的核心实现，负责工具的注册（懒加载元数据解析）、查找和执行，并通过事件分发系统（可选）提供工具调用生命周期的钩子。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly iterable $tools,                          // 工具对象集合
    private readonly ToolFactoryInterface $toolFactory,        // 工具元数据工厂（默认 ReflectionToolFactory）
    private readonly ToolCallArgumentResolverInterface $argumentResolver,  // 参数解析器
    private readonly LoggerInterface $logger,
    private readonly ?EventDispatcherInterface $eventDispatcher,
)
```

### getTools(): Tool[]

- 懒加载：首次调用时遍历所有工具对象，使用 `$toolFactory` 解析其元数据（`Tool` 对象），构建 `$toolsMetadata` 数组和 `$instanceMap`（工具名 → 实例对象的映射）。
- 后续调用直接返回缓存的 `$toolsMetadata`。

### execute(ToolCall $toolCall): ToolResult

完整执行流程：

1. **查找元数据**：`getMetadata()` 在 `$toolsMetadata` 中按名称查找，未找到则抛出 `ToolNotFoundException`。
2. **分发 `ToolCallRequested` 事件**：监听者可以拒绝执行（`deny()`）或直接设置结果（`setResult()`）以跳过实际调用。
3. **查找可执行对象**：`getExecutable()` 从 `$instanceMap` 或工具对象列表中获取实际执行对象。
4. **解析参数**：`$argumentResolver->resolveArguments()` 将 `ToolCall` 中的 JSON 参数反序列化为强类型 PHP 值。
5. **分发 `ToolCallArgumentsResolved` 事件**：参数已解析，即将调用工具。
6. **注入来源集合**：若工具实现了 `HasSourcesInterface`，创建新的 `SourceCollection` 并注入。
7. **执行工具**：调用 `$tool->{method}(...$arguments)`，包装为 `ToolResult`。
8. **分发 `ToolCallSucceeded` 或 `ToolCallFailed` 事件**。
9. **异常处理**：`ToolExecutionExceptionInterface` 直接重新抛出（供 `FaultTolerantToolbox` 捕获）；其他 `\Throwable` 包装为 `ToolExecutionException` 后抛出。

## 设计模式

- **注册表模式（Registry）**：通过 `$instanceMap` 维护工具名到实例的映射。
- **懒加载（Lazy Loading）**：`$toolsMetadata` 首次访问时才初始化，节省不必要的反射开销。
- **观察者模式（Observer）**：通过 `EventDispatcherInterface` 在工具调用各阶段发布事件，允许外部监听和干预。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolboxInterface` | 实现此接口 |
| `FaultTolerantToolbox` | 包装此类以捕获异常 |
| `ToolFactoryInterface` | 用于解析工具元数据 |
| `ToolCallArgumentResolverInterface` | 用于解析工具调用参数 |
| `ToolResult` | `execute()` 的返回类型 |
| `Event/ToolCallRequested` | 工具执行前的可中断事件 |
| `Event/ToolCallSucceeded` | 工具执行成功后的事件 |
| `Event/ToolCallFailed` | 工具执行失败后的事件 |
| `Exception/ToolNotFoundException` | 找不到工具时抛出 |
| `Exception/ToolExecutionException` | 工具执行异常时包装抛出 |
