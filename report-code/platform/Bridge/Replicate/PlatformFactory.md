# `Replicate/PlatformFactory.php` 分析

## 概述

Replicate Bridge 的平台工厂，将 `Client`（含轮询逻辑）、`LlamaModelClient`、`LlamaResultConverter` 和 `LlamaMessageBagNormalizer` 组装为完整的 `Platform` 实例。

## 关键方法

| 方法 | 说明 |
|------|------|
| `create(apiKey, httpClient, modelCatalog, contract, eventDispatcher): Platform` | 创建 `Client`（使用 `Clock` 和 `HttpClient`），组装 Platform |

## 设计要点

- 默认 `httpClient` 使用 `HttpClient::create()`（非 `EventSourceHttpClient`，因为 Replicate 不使用 SSE）
- 默认 `contract` 使用 `Contract::create(new LlamaMessageBagNormalizer())`，确保消息包被正确转换为 Llama prompt 格式
- 使用 `Symfony\Component\Clock\Clock` 作为实时时钟实例

## 关系

- 创建：`Client`（含 `Clock`）、`LlamaModelClient`、`LlamaResultConverter`
- 默认契约：`Contract`（注册 `LlamaMessageBagNormalizer`）
- 默认目录：`Replicate\ModelCatalog`
- 输出：`Platform`
