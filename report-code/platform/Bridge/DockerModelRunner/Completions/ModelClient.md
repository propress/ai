# Completions/ModelClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner\Completions`

## 概述

`ModelClient` 实现 `ModelClientInterface`，向 Docker Model Runner 的 `/engines/v1/chat/completions` 端点发送 POST 请求，要求 payload 必须为数组（不接受字符串），使用 `EventSourceHttpClient` 支持流式响应。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Completions` 实例。
- `request(Model $model, array|string $payload, array $options): RawHttpResult` — 合并 `$options` 和 `$payload` 数组发送 JSON 请求；payload 为字符串时抛出 `InvalidArgumentException`。

## 设计模式

- **强类型 payload 校验**：构造时将 `HttpClientInterface` 包装为 `EventSourceHttpClient`，确保支持 SSE 流式响应。

## 关联关系

- 由 `PlatformFactory` 注入 `$httpClient` 和 `$hostUrl`。
- 返回 `RawHttpResult`，由 `Completions\ResultConverter` 消费。
