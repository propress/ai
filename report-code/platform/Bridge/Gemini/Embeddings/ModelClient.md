# Embeddings/ModelClient.php 分析报告

## 文件概述

`Embeddings\ModelClient` 向 Gemini 的 `batchEmbedContents` 端点发送批量文本嵌入请求，支持维度控制和任务类型指定。

## 关键方法分析

### `request(Model $model, array|string $payload, array $options = []): RawHttpResult`

**URL：**
```
https://generativelanguage.googleapis.com/v1beta/models/{name}:batchEmbedContents
```

**请求体构造：**
```php
'requests' => array_map(
    static fn(string $text) => array_filter([
        'model'               => 'models/' . $model->getName(),
        'content'             => ['parts' => [['text' => $text]]],
        'outputDimensionality' => $modelOptions['dimensions'] ?? null,
        'taskType'             => $modelOptions['task_type'] ?? null,
        'title'                => $options['title'] ?? null,
    ]),
    is_array($payload) ? $payload : [$payload],
)
```

**鉴权：** `x-goog-api-key` 请求头（与聊天 Client 相同）

## 设计特点

- `array_filter` 自动移除 null 值，未设置的可选参数不出现在请求体中
- 每个 request 条目必须包含 `model` 字段（`models/` 前缀格式），这是 `batchEmbedContents` API 的特殊要求
- `payload` 支持单字符串（自动包装为单元素数组）或字符串数组（批量）
- `title` 从运行时 `$options` 读取（非模型选项），用于文档检索场景的标题元数据

## 与 VertexAi Embeddings ModelClient 的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| API 端点 | `batchEmbedContents` | `predict` |
| 鉴权 | API Key 请求头 | OAuth2 / API Key 查询参数 |
| 请求体格式 | `requests[]` 数组 | `instances[]` 数组 |
| 模型字段 | 每个 request 含 `model` | 无 |

## 关联文件

- `Embeddings/ResultConverter.php` — 解析响应中的 `embeddings[].values`
- `Embeddings/TaskType.php` — 提供 `task_type` 合法值
- `Embeddings.php` — 标记类，`supports()` 检查 `instanceof Embeddings`
