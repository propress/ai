# Toolbox/ToolFactoryInterface.php 分析报告

## 概述

`ToolFactoryInterface` 定义了工具元数据工厂的契约：给定一个工具对象或类名，返回对应的 `Tool` 元数据对象可迭代集合（一个类可通过重复 `#[AsTool]` 注解暴露多个工具方法）。

## 关键方法

### getTool(object|string $reference): iterable<Tool>

- `$reference` 可以是工具对象实例或类名字符串。
- 返回 `iterable<Tool>`，允许一个类注册多个工具（如 `Filesystem` 注册了 10 个工具操作）。
- 找不到有效工具定义时抛出 `ToolException`。

## 内置实现

| 实现类 | 策略 |
|---|---|
| `ReflectionToolFactory` | 通过反射读取 `#[AsTool]` 注解解析工具元数据（默认） |
| `MemoryToolFactory` | 通过编程式 `addTool()` 注册，无需注解 |
| `ChainFactory` | 依次尝试多个工厂，取第一个成功的结果 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox` | 构造注入此接口，在 `getTools()` 中调用 |
| `ToolFactory/ReflectionToolFactory` | 基于注解的默认实现 |
| `ToolFactory/MemoryToolFactory` | 编程式注册实现 |
| `ToolFactory/ChainFactory` | 工厂链实现 |
