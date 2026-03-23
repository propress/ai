# Toolbox/Exception/ToolConfigurationException.php 分析报告

## 概述

`ToolConfigurationException` 是 `ToolException` 的子类，专门表示工具配置错误（当前仅用于方法不存在的情况），继承自 `Agent\Exception\InvalidArgumentException`。

## 静态构造器

### invalidMethod(string $toolClass, string $methodName, ReflectionException $previous): self

工具类中指定的方法不存在时（`#[AsTool(method: 'nonExistentMethod')]`）抛出：
`'Method "..." not found in tool "...".'`

包装 `ReflectionException` 作为 `$previous` 保留原始异常上下文。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Exception/ToolException` | 继承此类 |
| `ToolFactory/ReflectionToolFactory` | `getTool()` 中方法不存在时抛出 |
| `ToolFactory/MemoryToolFactory` | `addTool()` 中方法不存在时抛出 |
