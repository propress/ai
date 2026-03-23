# Mistral/Embeddings/ModelClient.php

## 概述

`ModelClient`（位于 `Embeddings` 子命名空间）是 Mistral 嵌入 API 的 HTTP 请求客户端，发送 POST 请求至 `https://api.mistral.ai/v1/embeddings`。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient, string $apiKey)`
- 将传入的 `HttpClientInterface` 包装为 `EventSourceHttpClient`（若尚非此类型），以支持 SSE 流式响应（尽管嵌入 API 通常不使用流式）。
- API Key 参数标注 `#[\SensitiveParameter]`，防止泄露。

### `supports(Model $model): bool`
检查模型是否为 `Embeddings` 实例（`Bridge\Mistral\Embeddings` 顶层类）。

### `request(Model $model, array|string $payload, array $options = []): RawHttpResult`
构造请求：
- 端点：`https://api.mistral.ai/v1/embeddings`
- 认证：Bearer Token（`auth_bearer`）
- 请求体：`{model: '<name>', input: <payload>, ...$options}`

`$payload` 作为 `input` 字段传入（字符串或字符串数组均可），`$options` 中的其他参数通过 `array_merge` 合并到请求体。

## 设计模式

- **策略模式**：通过 `supports()` 分发到正确的客户端。
- 声明为 `final`，不允许继承。

## 关联关系

- 依赖顶层 `Bridge\Mistral\Embeddings` 类（而非自身命名空间中的任何类）进行类型检查。
- 返回 `RawHttpResult`，由同命名空间的 `ResultConverter` 消费。
- 由 `Mistral\PlatformFactory` 注册，优先级在 `Llm\ModelClient` 之前。
