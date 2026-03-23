# Toolbox/ToolFactory/MemoryToolFactory.php 分析报告

## 概述

`MemoryToolFactory` 是 `ToolFactoryInterface` 的编程式实现，允许在不使用 `#[AsTool]` 注解的情况下，通过代码调用 `addTool()` 动态注册工具，适用于动态工具列表或需要运行时配置工具元数据的场景。

## 关键方法分析

### addTool(string|object $class, string $name, string $description, string $method = '__invoke'): self

- 接受类名字符串或对象实例，以及工具名称、描述和入口方法。
- 使用对象时，以 `spl_object_id()` 作为键（避免同一类的不同实例互相覆盖）；使用类名字符串时，以类名作为键。
- 调用 `Factory::buildParameters()` 生成参数 Schema，构建 `Tool` 对象并追加到内部映射中。
- 若方法不存在，抛出 `ToolConfigurationException::invalidMethod()`。
- 返回 `$this`，支持链式调用。

### getTool(object|string $reference): iterable<Tool>

- 对象实例：先用 `spl_object_id()` 查找，若不存在则降级用类名查找（兼容按类名注册再按实例获取的场景）。
- 类名字符串：直接用类名查找。
- 未找到时抛出 `ToolException::invalidReference()`。

## 设计模式

- **建造者（Builder）**：`addTool()` 链式调用逐步构建工具注册表。
- **对象标识（Object Identity）**：使用 `spl_object_id()` 区分同一类的不同实例，允许同一类以不同描述注册为多个工具。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolFactoryInterface` | 实现此接口 |
| `ToolFactory/ReflectionToolFactory` | 对比：基于注解的静态工厂 |
| `ToolFactory/ChainFactory` | 可与本类组合：优先尝试注解，降级到编程式注册 |
| `Exception/ToolConfigurationException` | 方法不存在时抛出 |
| `Exception/ToolException` | 引用无效时抛出 |
