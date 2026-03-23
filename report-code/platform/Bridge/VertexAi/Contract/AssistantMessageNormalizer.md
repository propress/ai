# Contract/AssistantMessageNormalizer.php 分析报告

## 文件概述

将 `AssistantMessage` 序列化为 VertexAi Gemini API 的 `parts` 格式，处理文本回复和工具调用两种情形，字段格式与 Gemini Bridge 存在微小差异。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
$normalized = [];

if (null !== $data->getContent()) {
    $normalized[] = ['text' => $data->getContent()];  // 作为数组元素追加
}

if ($data->hasToolCalls()) {
    $normalized['functionCall'] = [
        'name' => $data->getToolCalls()[0]->getName(),  // 无 'id' 字段
        // 'args' 按需追加
    ];
}

return $normalized;  // 直接返回，不再包装为 [$normalized]
```

## 与 Gemini Bridge 对应类的关键差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| `functionCall.id` | ✅ 包含 | ❌ 不包含（VertexAi 不需要） |
| 文本 part 构建方式 | `$normalized['text'] = ...`（字段赋值） | `$normalized[] = ['text' => ...]`（数组追加） |
| 最终返回 | `[$normalized]`（额外包装） | `$normalized`（直接返回） |

## 设计特点

- VertexAi 的 `functionCall` 不含 `id` 字段，这与 VertexAi API 规范一致
- 文本和工具调用可以共存于同一个 `$normalized` 数组中（文本为索引元素，functionCall 为命名键）
- `supportsModel` 检查 `$model instanceof VertexAi\Gemini\Model`（而非 `Gemini\Gemini`）

## 关联文件

- `MessageBagNormalizer.php` — 调用此 Normalizer 处理助手消息
- `Gemini/ResultConverter.php` — 生成工具调用请求，由此 Normalizer 序列化历史消息
