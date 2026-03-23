# `Scaleway/Llm/ModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，向 Scaleway 聊天补全 API（`https://api.scaleway.ai/v1/chat/completions`）发送 HTTP POST 请求，内部使用 `EventSourceHttpClient` 以支持 SSE 流式输出。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Scaleway\Scaleway` 实例 |
| `request(Model, payload, options): RawHttpResult` | 使用 Bearer Token 认证，发送合并后的 JSON 请求体 |

## 设计要点

- 构造时将 `HttpClientInterface` 包装为 `EventSourceHttpClient`（若尚未包装），支持流式响应
- `payload` 必须为数组，否则抛出 `InvalidArgumentException`
- 使用 `auth_bearer` 认证，与 `Embeddings\ModelClient` 一致

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Scaleway\Scaleway`
- 被 `Scaleway\PlatformFactory` 实例化
- 对应转换器：`Llm\ResultConverter`
