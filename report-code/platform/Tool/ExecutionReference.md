# ExecutionReference 分析报告

## 文件概述
`ExecutionReference` 是工具执行目标的引用，包含 PHP 类名和方法名，作为指向实际工具实现的"指针"。

## 类定义
- **类型**: `final class`（值对象）

## 属性
- `$class`: `string` — 工具类的 FQCN
- `$method`: `string` — 执行方法名，默认 `'__invoke'`

## 方法
- `getClass(): string`
- `getMethod(): string`

## 设计模式
**值对象 + 命令标识符**：不持有对象实例，只持有类名和方法名字符串，允许序列化、跨请求传递，也允许通过 DI 容器按需实例化工具对象。

**为什么默认 `__invoke`**：工具类通常是单一职责的 invokable service（只有一个 `__invoke` 方法），这个默认值减少了大量样板代码。

## 与其他文件的关系
- 被 `Tool` 持有
- 被 Agent 模块的 `ToolExecutor` 使用（通过类名从容器获取实例，再调用指定方法）
