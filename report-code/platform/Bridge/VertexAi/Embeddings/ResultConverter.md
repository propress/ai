# Embeddings/ResultConverter.php 分析报告

## 文件概述

将 VertexAi `predict` API 的嵌入响应解析为 `VectorResult`，支持错误响应检测，并处理嵌套更深的响应结构（相比 Gemini Bridge）。

## 关键方法分析

### `convert(RawResultInterface $result, array $options = []): VectorResult`

```php
return new VectorResult(
    ...array_map(
        static fn(array $item): Vector => new Vector($item['embeddings']['values']),
        $data['predictions'],
    ),
);
```

响应结构期望（与 Gemini Bridge 不同）：
```json
{
  "predictions": [
    {"embeddings": {"values": [0.1, 0.2, ...], "statistics": {...}}},
    {"embeddings": {"values": [0.3, 0.4, ...]}}
  ]
}
```

**增强的错误处理（相比 Gemini Bridge 版本）：**
```php
if (isset($data['error'])) {
    throw new RuntimeException(
        sprintf('Error from Embeddings API: "%s"', $data['error']['message'] ?? 'Unknown error'),
        $data['error']['code']
    );
}
```

## 与 Gemini Bridge 对应类的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| 向量路径 | `embeddings[].values` | `predictions[].embeddings.values` |
| API 错误检查 | ❌ 无 | ✅ 检查 `error` 字段并设置错误码 |

## 关联文件

- `Embeddings/ModelClient.php` — 发送请求并返回原始响应
- `Embeddings/Model.php` — `supports()` 检查标记类
- `Symfony\AI\Platform\Vector\Vector` — 包装单个嵌入向量
