# Embeddings/ResultConverter.php 分析报告

## 文件概述

将 Gemini `batchEmbedContents` API 的响应解析为 `VectorResult`，包含批量嵌入向量列表。

## 关键方法分析

### `convert(RawResultInterface $result, array $options = []): VectorResult`

```php
return new VectorResult(
    ...array_map(
        static fn(array $item): Vector => new Vector($item['values']),
        $data['embeddings'],
    ),
);
```

响应结构期望：
```json
{
  "embeddings": [
    {"values": [0.1, 0.2, ...]},
    {"values": [0.3, 0.4, ...]}
  ]
}
```

**错误处理：** 响应无 `embeddings` 键时抛出 `RuntimeException('Response does not contain data.')`

### `getTokenUsageExtractor(): null`

嵌入请求无 Token 用量统计，始终返回 null。

## 与 VertexAi 对应类的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| 响应键 | `embeddings[].values` | `predictions[].embeddings.values` |
| 错误检查 | 仅检查键缺失 | 先检查 `error` 字段，再检查键缺失 |

## 关联文件

- `Embeddings/ModelClient.php` — 发送请求，返回原始响应
- `Embeddings.php` — `supports()` 检查标记类
- `Symfony\AI\Platform\Vector\Vector` — 包装单个嵌入向量的浮点数组
