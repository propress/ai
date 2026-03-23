# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia`

## 概述

`PlatformFactory` 是 Cartesia 桥接器的静态工厂类，通过 `create()` 方法接收 `apiKey` 和 `version` 参数，组装 `CartesiaClient`、`CartesiaResultConverter`、`ModelCatalog` 及 `CartesiaContract`，返回完整的 `Platform` 实例。

## 关键方法

- `create(string $apiKey, string $version, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform` — `apiKey` 和 `version` 为必填参数，其余均有默认值。

## 设计模式

- **静态工厂方法**：强制要求认证参数（`apiKey`、`version`），确保不会创建未认证的实例。

## 关联关系

- 组装 `CartesiaClient`（注入 `apiKey`、`version`）、`CartesiaResultConverter`、`ModelCatalog`（默认）、`CartesiaContract`（默认）。
- 返回 `Platform` 实例。
