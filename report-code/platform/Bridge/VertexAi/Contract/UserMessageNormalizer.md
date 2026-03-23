# Contract/UserMessageNormalizer.php 分析报告

## 文件概述

将 `UserMessage` 序列化为 VertexAi Gemini API 的 `parts` 数组格式，支持文本和 Base64 文件两种内容类型，字段命名使用 camelCase。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
// 文本内容
['text' => $content->getText()]

// 文件内容
['inlineData' => [
    'mimeType' => $content->getFormat(),  // camelCase
    'data'     => $content->asBase64(),
]]
```

## 与 Gemini Bridge 对应类的差异

| 字段 | Gemini（snake_case） | VertexAi（camelCase） |
|------|---------------------|----------------------|
| 文件字段键 | `inline_data` | `inlineData` |
| MIME 字段键 | `mime_type` | `mimeType` |

这是 Gemini REST API v1beta（snake_case）与 Vertex AI API（camelCase）之间的标准命名差异。

## 设计特点

- `supportsModel` 检查 `$model instanceof VertexAi\Gemini\Model`
- 逻辑与 Gemini 版本完全对称，仅字段名不同
- 不处理 `FileUrl` 等其他内容类型（仅 `Text` 和 `File`）

## 关联文件

- `MessageBagNormalizer.php` — 将此输出作为 `role: user` 消息的 `parts`
- `Symfony\AI\Platform\Message\Content\File` — 提供 `asBase64()` 和 `getFormat()`
