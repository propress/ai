# `Azure/Responses/ModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，向 Azure OpenAI Responses API 端点（`/openai/v1/responses`）发送请求，使用 `api-key` 头认证，支持通过 `deployment` 参数覆盖请求中的模型名称。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `OpenResponses\ResponsesModel` 实例 |
| `request(Model, payload, options): RawHttpResult` | POST 至预构建的 `$endpoint`，在 JSON 体中注入 `model` 字段（优先使用 `deployment` 参数） |

## 设计要点

- 端点在构造时一次性拼接并缓存为 `$this->endpoint`，格式为 `https://<baseUrl>/openai/v1/responses`
- `deployment` 参数为可选项（`?string`）：传入时覆盖模型名称，未传入时使用 `$model->getName()`
- 构造时校验 `baseUrl` 非空且不含协议前缀，`apiKey` 非空
- 使用 `EventSourceHttpClient` 包装器以支持潜在的 SSE 流式响应

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Bridge\OpenResponses\ResponsesModel`
- 被 `Azure\OpenAi\PlatformFactory` 实例化（传入 `deployment` 参数）
- 对应转换器：`Bridge\OpenResponses\ResultConverter`
