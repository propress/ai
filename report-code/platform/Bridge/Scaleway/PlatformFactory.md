# `Scaleway/PlatformFactory.php` 分析

## 概述

Scaleway Bridge 的平台工厂，同时注册 LLM（`Llm\ModelClient`）与嵌入向量（`Embeddings\ModelClient`）两套客户端及其对应转换器，构建支持双模型类型的完整 `Platform` 实例。

## 关键方法

| 方法 | 说明 |
|------|------|
| `create(apiKey, httpClient, modelCatalog, contract, eventDispatcher): Platform` | 实例化两个客户端、两个转换器，组装并返回 `Platform` |

## 设计要点

- 输入的 `httpClient` 被强制包装为 `EventSourceHttpClient`，以支持 LLM 的 SSE 流式输出
- `apiKey` 标注 `#[\SensitiveParameter]`，防止泄漏
- 与 `Azure\OpenAi\PlatformFactory` 不同，Scaleway 不强制使用特定 `contract`，默认传入 `null`

## 关系

- 创建：`Llm\ModelClient`、`Embeddings\ModelClient`、`Llm\ResultConverter`、`Embeddings\ResultConverter`
- 默认目录：`Scaleway\ModelCatalog`
- 输出：`Platform`（含两套模型处理管道）
