# Mistral/PlatformFactory.php

## 概述

`PlatformFactory` 是 Mistral Bridge 的统一入口点，提供 `create()` 静态工厂方法，将 HTTP 客户端、模型客户端、结果转换器和 Contract 规范化器组装为完整的 `Platform` 实例。

## 关键方法分析

### `static create(string $apiKey, ...): Platform`

参数说明：
- `$apiKey`：Mistral API Key，标注 `#[\SensitiveParameter]`，为唯一必填参数。
- `$httpClient`：可选，若提供非 `EventSourceHttpClient` 类型则自动包装，若为 `null` 则由内部创建。
- `$modelCatalog`：默认使用 `Mistral\ModelCatalog`。
- `$contract`：可选的自定义 Contract。
- `$eventDispatcher`：可选的事件分发器。

**两个 ModelClient 的注册顺序**：`Embeddings\ModelClient` 排在 `Llm\ModelClient` 之前，确保嵌入请求优先匹配到正确的客户端（尽管 `supports()` 已保证正确分发）。

**默认 Contract**：仅包含 Mistral 特有的四个规范化器（`ToolNormalizer`、`DocumentNormalizer`、`DocumentUrlNormalizer`、`ImageUrlNormalizer`），通用规范化器由 `Contract::create()` 基类自动注册。

## 设计模式

- **工厂方法模式**：静态工厂封装复杂的对象图构建。
- **HttpClient 自动包装**：与 `Bedrock\PlatformFactory` 不同，Mistral 在工厂层统一完成 `EventSourceHttpClient` 包装，两个客户端共享同一个包装后的实例。

## 关联关系

- 聚合 `Embeddings\ModelClient` + `Llm\ModelClient` 两个客户端。
- 聚合 `Embeddings\ResultConverter` + `Llm\ResultConverter` 两个转换器。
- 注册四个 Mistral 专有 Contract 规范化器。
