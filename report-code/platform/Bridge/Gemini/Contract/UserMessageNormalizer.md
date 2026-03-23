# Contract/UserMessageNormalizer.php 分析报告

## 文件概述

将 `UserMessage`（用户消息）序列化为 Gemini API 的 `parts` 数组格式，支持纯文本和 Base64 内联文件（多模态）两种内容类型。

## 关键方法分析

### `normalize(mixed $data, ...): array`

遍历消息的内容列表，按内容类型生成不同的 part：

```php
// 文本内容
['text' => $content->getText()]

// 文件内容（图像/音频/PDF/视频）
['inline_data' => [
    'mime_type' => $content->getFormat(),  // snake_case
    'data'      => $content->asBase64(),
]]
```

## 与 VertexAi 对应类的差异

| 字段命名 | Gemini | VertexAi |
|---------|--------|---------|
| 文件字段键 | `inline_data`（snake_case） | `inlineData`（camelCase） |
| MIME 字段键 | `mime_type`（snake_case） | `mimeType`（camelCase） |

这是两个桥接在 API 命名风格上的典型差异点：Gemini REST API v1beta 使用 snake_case，VertexAi Vertex AI API 使用 camelCase。

## 设计特点

- `$content->asBase64()` 将二进制文件转为 Base64 字符串，Gemini 通过 inline 方式传递多模态内容（非 URL 引用）
- 不支持 URL 类型的文件内容（如 `FileUrl`），仅处理 `File`（inline data）

## 关联文件

- `MessageBagNormalizer.php` — 将此 Normalizer 的输出作为 `role: user` 消息的 `parts`
- `Symfony\AI\Platform\Message\Content\File` — 提供 `asBase64()` 和 `getFormat()` 方法
- `Symfony\AI\Platform\Message\Content\Text` — 提供 `getText()` 方法
