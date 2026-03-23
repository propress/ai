# PlatformFactory.php 分析

## 概述

`PlatformFactory` 是 Perplexity Bridge 的静态工厂类，将 `ModelClient`、`ResultConverter`、`ModelCatalog`、`PerplexityContract` 组装为完整的 `Platform` 实例，提供一行代码初始化入口。

## 关键方法分析

### `create(string $apiKey, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`（静态）
- 将 HTTP 客户端包装为 `EventSourceHttpClient`（幂等检查），支持 SSE 流式。
- `$modelCatalog` 默认为 `new ModelCatalog()`（含 5 个 Sonar 模型）。
- `$contract` 默认为 `PerplexityContract::create()`（注册 `FileUrlNormalizer`）。
- 将构造好的组件注入 `Platform` 实例。

## 关键模式

- **静态工厂方法**：无需实例化工厂本身，适合框架外的快速初始化场景。
- **防御性包装**：`EventSourceHttpClient` 的幂等包装逻辑与其他 Bridge 一致，避免双重包装。
- **`#[\SensitiveParameter]`** 标注 API Key。

## 关联关系

- 是外部用户（或 AI Bundle 的 DI 配置）初始化 Perplexity Platform 的主要入口点。
- 组装 `ModelClient`、`ResultConverter`、`ModelCatalog`、`PerplexityContract` 四大组件。
