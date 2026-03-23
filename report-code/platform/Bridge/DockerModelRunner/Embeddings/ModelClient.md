# Embeddings/ModelClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner\Embeddings`

## 概述

`ModelClient` 实现 `ModelClientInterface`，向 Docker Model Runner 的 `/engines/v1/embeddings` 端点发送 POST 请求，将模型名称和输入内容合并为 JSON Body，无需特殊认证。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Embeddings` 实例。
- `request(Model $model, array|string $payload, array $options): RawHttpResult` — 构建包含 `model`（`$model->getName()`）和 `input`（`$payload`）的 JSON Body，合并额外选项。

## 设计模式

- **简洁 HTTP 请求**：不使用 `EventSourceHttpClient` 包装（嵌入不需要流式），直接使用注入的 `HttpClientInterface`。

## 关联关系

- 由 `PlatformFactory` 注入 `$httpClient` 和 `$hostUrl`。
- 返回 `RawHttpResult`，由 `Embeddings\ResultConverter` 消费。
