# PlatformFactory.php 分析报告

## 概述

`PlatformFactory` 是 Cerebras Bridge 的工厂类，直接实例化 `Platform` 并注册自定义 `ToolNormalizer`（通过 `Contract::create()`），是少数不依赖 `Generic\PlatformFactory` 而独立构建平台的 Bridge 之一。

## 关键方法分析

### `create(string $apiKey, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`

- 将 HTTP 客户端升级为 `EventSourceHttpClient`
- 注册 `[new ModelClient($httpClient, $apiKey)]` 作为 ModelClient 列表
- 注册 `[new ResultConverter()]` 作为 ResultConverter 列表
- **关键**：当 `$contract` 未提供时，使用 `Contract::create(new ToolNormalizer())` 创建含自定义序列化器的契约，确保工具定义包含 `parameters` 字段
- 默认模型目录为 `new ModelCatalog()`

## 设计模式

- **显式构建（Explicit Construction）**：直接 `new Platform([...])` 而非委托 Generic 层，获得对所有组件的完整控制权
- **默认安全契约**：通过 `$contract ?? Contract::create(new ToolNormalizer())` 确保在无自定义契约时仍应用兼容性修复

## 与其他类的关系

- 创建并注入 `Cerebras\ModelClient` 和 `Cerebras\ResultConverter`（均非 Generic 版本）
- 自定义 `Contract` 包含 `Cerebras\Contract\ToolNormalizer`，解决 Cerebras API 的工具参数兼容性问题
- 与 `AmazeeAi\PlatformFactory` 同为"显式构建"模式，与 Albert/OpenRouter 的"委托 Generic"模式形成对比
