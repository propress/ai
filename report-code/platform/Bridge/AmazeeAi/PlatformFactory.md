# PlatformFactory.php 分析报告

## 概述

`PlatformFactory` 组装 AmazeeAi Bridge 的 `Platform` 实例，使用 Generic Bridge 的 HTTP ModelClient 与自定义的 `CompletionsResultConverter`，并将 `EventSourceHttpClient` 包装应用于流式响应支持。

## 关键方法分析

### `create(string $baseUrl, ?string $apiKey, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`

- 将传入的 HTTP 客户端升级为 `EventSourceHttpClient` 以支持 SSE 流式响应
- 注册两个 ModelClient：`Generic\Completions\ModelClient`（聊天补全）和 `Generic\Embeddings\ModelClient`（嵌入向量）
- 注册两个 ResultConverter：`AmazeeAi\CompletionsResultConverter`（含 LiteLLM 兼容修复）和 `Generic\Embeddings\ResultConverter`
- 默认模型目录为 `Generic\FallbackModelCatalog`（透传任意模型名，适合动态模型场景）

## 设计模式

- **静态工厂方法**：无需实例化工厂类即可创建 Platform
- **显式组件注册**：直接通过 `new Platform([...clients], [...converters], ...)` 构建，与 Albert/OpenRouter 不同（后者委托给 `GenericPlatformFactory`），提供更精细的控制
- **可选认证**：`$apiKey` 为可选参数，支持访问无认证的公开端点

## 与其他类的关系

- 直接实例化 `Platform` 而非委托 `Generic\PlatformFactory`，明确控制使用哪个 `ResultConverter`
- 默认使用 `Generic\FallbackModelCatalog`，可替换为 `AmazeeAi\ModelApiCatalog` 获得动态模型发现能力
