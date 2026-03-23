# Mistral/Llm/ModelClient.php

## 概述

`ModelClient`（位于 `Llm` 子命名空间）是 Mistral 对话 API 的 HTTP 请求客户端，发送 POST 请求至 `https://api.mistral.ai/v1/chat/completions`，支持标准请求和流式响应。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient, string $apiKey)`
与 `Embeddings\ModelClient` 相同：将 HTTP 客户端包装为 `EventSourceHttpClient` 以支持 SSE 流式响应，API Key 参数标注 `#[\SensitiveParameter]`。

### `supports(Model $model): bool`
检查模型是否为 `Mistral` 实例（而非 `Embeddings`）。

### `request(Model $model, array|string $payload, array $options = []): RawHttpResult`
- **payload 类型验证**：若为字符串则抛出 `InvalidArgumentException`（对话 API 需要结构化的消息数组）。
- 将 `$options` 与 `$payload` 通过 `array_merge` 合并后作为 JSON 请求体发送。
- 包含两个请求头：`Content-Type: application/json` 和 `Accept: application/json`（嵌入客户端只有 `Content-Type`）。

## 设计模式

- **策略模式**：通过 `supports()` 分发。
- 两个 Accept 头的存在提示该端点可能在内容协商上有特殊要求。

## 关联关系

- 依赖顶层 `Bridge\Mistral\Mistral` 类进行类型检查。
- 返回 `RawHttpResult`，由 `Llm\ResultConverter` 消费。
- 与 `Embeddings\ModelClient` 共享 `EventSourceHttpClient` 包装逻辑（相同代码，不同类型检查）。
