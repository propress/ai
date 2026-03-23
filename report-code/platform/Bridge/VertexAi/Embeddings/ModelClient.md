# Embeddings/ModelClient.php 分析报告

## 文件概述

`VertexAi\Embeddings\ModelClient` 向 Vertex AI 的 `predict` 端点发送批量文本嵌入请求，同样支持项目端点和全局端点双模式，请求体格式遵循 VertexAi `predict` API 规范。

## 关键方法分析

### `request(BaseModel $model, array|string $payload, array $options = []): RawHttpResult`

**URL 构造（双端点模式）：**
```php
// 项目端点
"https://aiplatform.googleapis.com/v1/projects/{$projectId}/locations/{$location}/publishers/google/models/{$name}:predict"

// 全局端点
"https://aiplatform.googleapis.com/v1/publishers/google/models/{$name}:predict"
```

**请求体格式（与 Gemini Bridge 的 batchEmbedContents 完全不同）：**
```php
'instances' => array_map(
    static fn(string $text) => [
        'content'   => $text,                               // 直接字符串，非 parts 格式
        'title'     => $options['title'] ?? null,
        'task_type' => $modelOptions['task_type'] ?? TaskType::RETRIEVAL_QUERY,  // 有默认值
    ],
    ...
)
```

**鉴权：** API Key 放入 `query['key']`（查询参数），而非请求头

## 与 Gemini Bridge Embeddings ModelClient 的差异

| 方面 | Gemini Bridge | VertexAi |
|------|--------------|---------|
| API 端点方法 | `batchEmbedContents` | `predict` |
| 请求体顶层键 | `requests[]` | `instances[]` |
| 文本字段 | `content.parts[].text` | `content`（直接字符串） |
| 每项含 `model` 字段 | ✅ 是 | ❌ 否 |
| `task_type` 默认值 | 无（null 时 API 自选） | `RETRIEVAL_QUERY` |
| 鉴权位置 | 请求头 | 查询参数 |
| 额外模型选项 | 仅 `dimensions`/`task_type` | 其余 `modelOptions` 也合并到请求体 |

## 关联文件

- `Embeddings/ResultConverter.php` — 解析 `predictions[].embeddings.values`
- `Embeddings/TaskType.php` — 提供 `task_type` 默认值 `RETRIEVAL_QUERY`
- `PlatformFactory.php` — 实例化并传入 location/projectId/apiKey
