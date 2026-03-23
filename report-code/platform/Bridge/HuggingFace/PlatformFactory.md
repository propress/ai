# PlatformFactory.php 分析

## 概述

`PlatformFactory` 是 HuggingFace Bridge 的静态工厂类，将 `ModelClient`、`ResultConverter`、`ModelCatalog`、`HuggingFaceContract` 组装为可用的 `Platform` 实例，提供简洁的一行代码初始化入口。

## 关键方法分析

### `create(string $apiKey, string $provider, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`（静态）
- `$provider` 默认为 `Provider::HF_INFERENCE`，可切换至任意第三方推理提供商。
- 自动将传入的 `HttpClientInterface` 包装为 `EventSourceHttpClient`（支持 SSE 流式响应），若已是 `EventSourceHttpClient` 则直接使用。
- `$contract` 默认为 `HuggingFaceContract::create()`（注册 `FileNormalizer` 和 `MessageBagNormalizer`）。

## 关键模式

- **静态工厂方法**：无需实例化工厂本身，直接通过类名调用。
- **`#[\SensitiveParameter]`** 标注 `$apiKey`，防止在错误报告中泄露。
- **防御性 HttpClient 包装**：幂等的 `instanceof EventSourceHttpClient` 检查确保不会双重包装。

## 关联关系

- 组装 `ModelClient`、`ResultConverter`、`ModelCatalog`（默认）、`HuggingFaceContract`（默认）。
- 是外部用户初始化 HuggingFace Platform 的主要入口点。
