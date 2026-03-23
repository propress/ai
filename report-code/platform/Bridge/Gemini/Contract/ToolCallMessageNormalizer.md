# Contract/ToolCallMessageNormalizer.php 分析报告

## 文件概述

将 `ToolCallMessage`（工具调用结果消息）序列化为 Gemini API 的 `functionResponse` 格式，处理 JSON 字符串与非 JSON 内容的兼容性。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
return [[
    'functionResponse' => array_filter([
        'id'       => $data->getToolCall()->getId(),
        'name'     => $data->getToolCall()->getName(),
        'response' => is_array($resultContent) ? $resultContent : ['rawResponse' => $resultContent],
    ]),
]];
```

**核心处理逻辑：**

1. 尝试用 `json_validate()` 检测内容是否为 JSON，若是则 `json_decode` 解析
2. Gemini API 要求 `response` 字段为对象（object），若解析结果非数组，则包装为 `['rawResponse' => $value]`
3. 包含 `id` 字段（VertexAi 对应类不含此字段）
4. `array_filter` 移除 null 值（`id` 为空字符串时不过滤）

## 设计特点

- **防御性包装**：`['rawResponse' => $value]` 确保 Gemini 始终收到 object 类型，不会因纯字符串导致 API 报错
- `json_validate()` 是 PHP 8.3+ 内置函数，比 `json_decode` + 错误检查更高效
- 与 VertexAi 同名类的差异：Gemini 版本含 `id`，VertexAi 版本不含

## 关联文件

- `MessageBagNormalizer.php` — 将此 Normalizer 的输出作为 parts 放入 `role: user` 消息
- `Gemini/ResultConverter.php` — 生成工具调用请求（`ToolCallResult`），对应此处的响应
