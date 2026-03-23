# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`PlatformFactory` 是 ModelsDev Bridge 的核心工厂，通过 `create()` 方法为任意 models.dev 供应商创建 Platform 实例：优先检查是否需要专用桥接器（如 Anthropic、Gemini），若是则委托对应工厂；否则使用通用 Generic Bridge，自动解析 API 基础 URL 并检测供应商的补全/嵌入能力。

## 关键方法

- `create(string $provider, ?string $apiKey, ?string $baseUrl, ?string $dataPath, ?Contract $contract, ?HttpClientInterface $httpClient, ?EventDispatcherInterface $eventDispatcher): Platform` — 完整的路由逻辑：加载数据 → 检查专用桥接器 → 解析 API URL → 检测能力 → 创建 Generic Platform。

## 设计模式

- **自动路由策略**：`$baseUrl` 参数可强制使用 Generic Bridge（适合 OpenAI 兼容代理），覆盖专用桥接器检测。
- **能力自动检测**：扫描模型目录中的类类型来判断供应商是否支持补全和/或嵌入。

## 关联关系

- 依赖 `DataLoader`、`BridgeResolver`、`ProviderRegistry`、`ModelCatalog`（动态创建）。
- 委托 `GenericPlatformFactory` 或专用桥接器工厂完成最终 Platform 构建。
