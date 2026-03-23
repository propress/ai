# Toolbox/ToolResultConverter.php 分析报告

## 概述

`ToolResultConverter` 将 `ToolResult` 中工具返回的任意类型结果转换为字符串，以便作为工具调用结果消息发送给 LLM。

## 关键方法分析

### convert(ToolResult $toolResult): ?string

转换逻辑按优先级依次尝试：

1. `null`：直接返回 `null`。
2. `string`：直接返回原字符串。
3. `Stringable`：调用 `(string) $result`。
4. 其他类型（数组、对象等）：使用 Symfony Serializer 序列化为 JSON 字符串。

序列化失败时抛出 `RuntimeException`（非 Toolbox 异常，意味着不会被 `FaultTolerantToolbox` 捕获）。

### 默认 Serializer 配置

```php
new Serializer(
    [new JsonSerializableNormalizer(), new DateTimeNormalizer(), new ObjectNormalizer()],
    [new JsonEncoder()]
)
```

支持 `JsonSerializable` 对象、`DateTime`/`DateTimeImmutable` 以及普通 PHP 对象。

## 设计模式

- **转换器（Converter）**：单一职责，专门处理类型转换，将复杂类型化结果规整化为协议级字符串。
- **可替换序列化器**：通过构造函数注入 `SerializerInterface`，便于在 Symfony DI 容器中替换为已配置的全局序列化器实例。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentProcessor` | 持有 `ToolResultConverter` 实例，在迭代循环中调用 `convert()` |
| `ToolResult` | 转换的输入对象 |
| `Exception/RuntimeException` | 序列化失败时抛出 |
