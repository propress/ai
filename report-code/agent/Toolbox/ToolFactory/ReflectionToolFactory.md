# Toolbox/ToolFactory/ReflectionToolFactory.php 分析报告

## 概述

`ReflectionToolFactory` 是 `ToolFactoryInterface` 的默认实现，通过 PHP 反射读取工具类上的 `#[AsTool]` 注解，结合 Platform 模块的 JSON Schema 工厂（`Factory`）生成工具的完整元数据（`Tool` 对象，含参数 Schema）。

## 关键方法分析

### getTool(object|string $reference): iterable<Tool>

1. 获取类名（对象取 `::class`，字符串直接使用）。
2. 检查类是否存在，否则抛出 `ToolException::invalidReference()`。
3. 反射该类，获取所有 `#[AsTool]` 属性实例（可能有多个）。
4. 若无 `#[AsTool]` 属性，抛出 `ToolException::missingAttribute()`。
5. 对每个注解实例，调用 `$this->factory->buildParameters($className, $asTool->method)` 生成参数 Schema，构建 `Tool(ExecutionReference, name, description, parameters)` 对象，通过 `yield` 返回。
6. 若方法不存在（`ReflectionException`），抛出 `ToolConfigurationException::invalidMethod()`。

## 设计模式

- **工厂方法（Factory Method）**：封装对象创建逻辑，调用方无需了解反射和 JSON Schema 生成细节。
- **生成器（Generator）**：`yield` 支持延迟生成，适合一个类有多个工具方法的场景。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolFactoryInterface` | 实现此接口 |
| `Toolbox/Attribute/AsTool` | 读取此注解 |
| `Platform/Contract/JsonSchema/Factory` | 生成方法参数的 JSON Schema |
| `Exception/ToolException` | 类不存在或缺少注解时抛出 |
| `Exception/ToolConfigurationException` | 方法不存在时抛出 |
