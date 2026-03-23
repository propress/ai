# ToolCall 分析报告

## 文件概述
`ToolCall` 表示模型请求调用的单个工具，包含调用 ID、函数名和参数，实现 `JsonSerializable` 支持序列化为标准 OpenAI 格式。

## 类定义
- **类型**: `final class`，实现 `JsonSerializable`

## 方法分析

### `__construct(string $id, string $name, array $arguments)`
- `$id`：工具调用唯一 ID（如 `call_abc123`）
- `$name`：被调用函数名
- `$arguments`：已解析的参数数组

### `getId() / getName() / getArguments()`
- 标准 getter 方法

### `jsonSerialize(): array`
- 输出 OpenAI 标准格式：`{id, type: 'function', function: {name, arguments: json_string}}`
- **技巧**：空参数使用 `new \stdClass()` 而非 `[]`，确保序列化为 `{}` 而非 `[]`

## 设计模式
**值对象 + JsonSerializable**：不可变 DTO，内置序列化逻辑，用于在 ToolCallMessage 中回传给 AI 模型。

## 与其他文件的关系
- 被 `ToolCallResult` 聚合（一次响应可含多个工具调用）
- 被 `ToolCallMessage` 引用（作为工具执行结果的来源）
- 被 `ToolCallNormalizer` 序列化

## 使用示例
```php
$toolCall = new ToolCall('call_abc', 'get_weather', ['city' => 'Beijing']);
echo $toolCall->getName(); // get_weather
```
