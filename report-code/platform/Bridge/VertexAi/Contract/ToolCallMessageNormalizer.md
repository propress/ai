# Contract/ToolCallMessageNormalizer.php 分析报告

## 文件概述

将 `ToolCallMessage`（工具调用结果）序列化为 VertexAi Gemini API 的 `functionResponse` 格式，与 Gemini Bridge 版本相比不包含 `id` 字段，并使用更严格的 JSON 解析。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
return [[
    'functionResponse' => array_filter([
        'name'     => $data->getToolCall()->getName(),
        // 注意：无 'id' 字段（与 Gemini Bridge 不同）
        'response' => is_array($resultContent) ? $resultContent : ['rawResponse' => $resultContent],
    ]),
]];
```

**JSON 解析差异：**
- 使用 `json_decode(..., true, 512, \JSON_THROW_ON_ERROR)`（抛出异常）
- Gemini 版本使用 `json_decode(...)` 不抛出异常

## 与 Gemini Bridge 对应类的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| `id` 字段 | ✅ 包含 | ❌ 不含 |
| JSON 解析标志 | 无 `JSON_THROW_ON_ERROR` | 使用 `JSON_THROW_ON_ERROR` |
| `@throws` 声明 | 无 | `@throws \JsonException` |

## 设计特点

- `['rawResponse' => $value]` 包装逻辑与 Gemini 版本一致：确保 `response` 为 object 类型
- `array_filter` 移除 null 值

## 关联文件

- `MessageBagNormalizer.php` — 将此输出作为 `role: user` 消息的 parts
- `Gemini/ResultConverter.php` — 生成 `ToolCallResult`，触发工具调用流程
