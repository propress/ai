# PlatformSubscriber 分析报告

## 文件概述
`PlatformSubscriber` 是结构化输出功能的**事件驱动入口**，通过订阅 `InvocationEvent` 和 `ResultEvent`，在调用链前后注入结构化输出的 Schema 格式要求和结果反序列化逻辑，无需修改 `Platform` 核心代码。

## 类定义
- **类型**: `final class`，实现 `EventSubscriberInterface`

## 方法分析

### `getSubscribedEvents(): array`
订阅两个事件：
- `InvocationEvent` → `processInput()`
- `ResultEvent` → `processResult()`

### `processInput(InvocationEvent $event): void`
触发条件：options 中包含 `response_format` 键（值为类名字符串或对象实例）
1. 提取目标类名（支持已有对象的 populate 模式）
2. 检查不与 `stream: true` 同时使用（抛出 `InvalidArgumentException`）
3. 检查模型支持 `OUTPUT_STRUCTURED`（抛出 `MissingModelSupportException`）
4. 调用 `ResponseFormatFactory::create()` 生成 JSON Schema 格式定义
5. 将原始类名替换为 Schema 定义，写回 `event->setOptions()`

### `processResult(ResultEvent $event): void`
1. 取出当前 `DeferredResult` 的 `ResultConverter`
2. 创建新的 `StructuredOutput\ResultConverter`（包装原始 Converter）
3. 替换 `DeferredResult` 为使用新 Converter 的版本

**技巧**：通过**包装（Decorator）**原始 ResultConverter，不替换而是叠加反序列化能力，保留原始 Converter 的 Token 使用量提取逻辑。

## 设计模式
**事件订阅者（Event Subscriber）**：完全基于事件，不侵入 Platform 核心，可以随时通过取消注册此 Subscriber 禁用结构化输出功能。

**装饰器（Decorator）**：`processResult` 中包装原始 ResultConverter，而非替换，保留原有行为并叠加新行为。

## 扩展点
可以子类化或替换 `ResponseFormatFactory` 以支持不同的 Schema 格式（如 Anthropic 的 `tool_use` 格式而非 `json_schema`）。

## 与其他文件的关系
- 订阅 `Event/InvocationEvent` 和 `Event/ResultEvent`
- 使用 `ResponseFormatFactory` 生成 Schema
- 创建 `StructuredOutput\ResultConverter` 包装原 Converter
- 在 AI Bundle 中通过 `@autoconfigure` 自动注册为事件订阅者
