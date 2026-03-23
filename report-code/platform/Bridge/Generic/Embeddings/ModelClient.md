# Generic/Embeddings/ModelClient 分析报告

## 文件概述
`Generic\Embeddings\ModelClient` 是 OpenAI 兼容 `/v1/embeddings` 端点的通用 HTTP 客户端。

## 请求格式
```json
{"model": "text-embedding-3-small", "input": <payload>}
```
`input` 直接传入 `$payload`（字符串或字符串数组），支持批量嵌入。

## 与其他文件的关系
- 检查 `instanceof EmbeddingsModel`
- 被 `Generic\PlatformFactory`（`supportsEmbeddings: true`）注册
