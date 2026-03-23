# PlatformFactory.php 分析报告

## 概述

`PlatformFactory` 是 OpenRouter Bridge 的入口工厂类，将实际 Platform 构建委托给 `Generic\PlatformFactory`，仅需提供 OpenRouter 的固定 API 基础地址 `https://openrouter.ai/api`。

## 关键方法分析

### `create(string $apiKey, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`

- 将 HTTP 客户端升级为 `EventSourceHttpClient`（支持 SSE 流式响应）
- 调用 `Generic\PlatformFactory::create()`，传入固定 `baseUrl: 'https://openrouter.ai/api'` 和用户 API Key
- 默认模型目录为 `ModelCatalog`（静态），可替换为 `ModelApiCatalog` 获取动态模型列表
- `$contract` 参数可选，传入 null 使用 Generic 默认契约

## 设计模式

- **静态工厂方法**：`create()` 为静态，零配置即可使用
- **委托模式**：封装 OpenRouter 的固定端点地址，将复杂构建逻辑委托给 Generic 层
- **最小化配置**：与 Albert Bridge 不同，不做任何前置校验，职责专注于组合配置

## 与其他类的关系

- 委托 `Generic\PlatformFactory`，与 Albert、OpenRouter 同属"委托型"工厂
- 使用 `Generic` 层的 `Completions\ModelClient` 和 `Embeddings\ModelClient` 处理实际 HTTP 通信
- `$apiKey` 传递给 Generic ModelClient，以 `Authorization: Bearer` 头部方式发送
