# Contract/Normalizer/Result 目录分析报告

## 目录职责

`Contract/Normalizer/Result/` 目录包含结果类型的 Normalizer，用于序列化工具调用等结果对象。

**目录路径**: `src/platform/src/Contract/Normalizer/Result/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `ToolCallNormalizer.php` | 工具调用 Normalizer |

---

## ToolCallNormalizer

将 `ToolCall` 对象序列化为 API 格式：

```php
// 输入
new ToolCall('call_123', 'get_weather', ['location' => 'Paris']);

// 输出
[
    'id' => 'call_123',
    'type' => 'function',
    'function' => [
        'name' => 'get_weather',
        'arguments' => '{"location":"Paris"}'
    ]
]
```

---

## 使用场景

当 AssistantMessage 包含工具调用时，Normalizer 链会递归序列化：

```php
$assistantMessage = Message::ofAssistant(null, [
    new ToolCall('call_1', 'search', ['query' => 'AI']),
    new ToolCall('call_2', 'calculate', ['expr' => '2+2']),
]);

// 序列化后
[
    'role' => 'assistant',
    'content' => null,
    'tool_calls' => [
        [
            'id' => 'call_1',
            'type' => 'function',
            'function' => ['name' => 'search', 'arguments' => '{"query":"AI"}']
        ],
        [
            'id' => 'call_2',
            'type' => 'function',
            'function' => ['name' => 'calculate', 'arguments' => '{"expr":"2+2"}']
        ]
    ]
]
```
