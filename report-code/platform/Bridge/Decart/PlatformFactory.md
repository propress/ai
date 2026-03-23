# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart`

## 概述

`PlatformFactory` 是 Decart 桥接器的静态工厂类，通过 `create()` 方法接收必填的 `$apiKey` 参数和可选的 `$hostUrl`，组装 `DecartClient`、`DecartResultConverter`、`ModelCatalog` 及 `DecartContract`，返回完整的 `Platform` 实例。

## 关键方法

- `create(string $apiKey, ?string $hostUrl, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform` — `$apiKey` 标注 `#[SensitiveParameter]`，`$hostUrl` 默认为 `'https://api.decart.ai/v1'`。

## 设计模式

- **可覆盖端点**：`$hostUrl` 参数允许覆盖默认 API 地址，便于本地测试或私有部署。

## 关联关系

- 组装 `DecartClient`（注入 `$apiKey`、`$hostUrl`）、`DecartResultConverter`、`ModelCatalog`（默认）、`DecartContract`（默认）。
- 返回 `Platform` 实例。
