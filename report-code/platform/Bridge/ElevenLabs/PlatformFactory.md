# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`PlatformFactory` 是 ElevenLabs 桥接器的静态工厂类，通过 `create()` 方法组装 `ElevenLabsClient`、`ElevenLabsResultConverter`、模型目录（静态或动态）及 `ElevenLabsContract`，并使用 `ScopingHttpClient` 将 API Key 自动注入所有请求。

## 关键方法

- `create(string $endpoint, ?string $apiKey, ?HttpClientInterface $httpClient, bool $apiCatalog, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform` — 包装 HTTP 客户端、注入认证 Header、选择模型目录（静态/动态），返回 `Platform` 实例。

## 设计模式

- **双目录策略**：`$apiCatalog=true` 时使用 `ElevenLabsApiCatalog`，否则使用静态 `ModelCatalog`。
- **`ScopingHttpClient`**：将 `xi-api-key` Header 限定注入到指定 `$endpoint` 的所有请求。

## 关联关系

- 依赖 `EventSourceHttpClient`（流式支持）和 `ScopingHttpClient`（认证注入）。
- 组装并返回完整的 `Platform` 实例。
