# Generic/Embeddings/ResultConverter 分析报告

## 文件概述
解析 OpenAI 格式嵌入响应，将 `data[].embedding` 转换为 `VectorResult`（包含一或多个 `Vector`）。

## 响应格式解析
```json
{"data": [{"embedding": [0.1, 0.2, ...], "index": 0}]}
```
每个 `embedding` 数组 → `new Vector($floatArray)`。

## 错误处理
- 401 → `AuthenticationException`
- 400/404 → `BadRequestException`
- 429 → `RateLimitExceededException`（附错误消息）
- 无 `data[0].embedding` → `RuntimeException`

`getTokenUsageExtractor()` 返回 `null`（嵌入不计 Token 使用量）。
