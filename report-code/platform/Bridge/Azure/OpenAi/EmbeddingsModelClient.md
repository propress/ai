# `Azure/OpenAi/EmbeddingsModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，向 Azure OpenAI 嵌入向量端点发送 HTTP POST 请求，使用 `api-key` 请求头认证，端点 URL 含 deployment 名称和 `api-version` 查询参数。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Generic\EmbeddingsModel` 实例 |
| `request(Model, payload, options): RawHttpResult` | 发送至 `https://<baseUrl>/openai/deployments/<deployment>/embeddings?api-version=<version>`，使用 `api-key` 头认证 |

## 设计要点

- 构造时严格校验：`baseUrl` 不得含协议前缀、`deployment`/`apiVersion`/`apiKey` 不得为空字符串，违反时抛出 `InvalidArgumentException`
- 内部将 `HttpClientInterface` 包装为 `EventSourceHttpClient`
- 请求体中显式传入 `model` 名称和 `input` 字段

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Bridge\Generic\EmbeddingsModel`
- 被 `Azure\OpenAi\PlatformFactory` 实例化
