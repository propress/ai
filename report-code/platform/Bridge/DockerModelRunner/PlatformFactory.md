# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner`

## 概述

`PlatformFactory` 是 DockerModelRunner 桥接器的静态工厂类（非 final），通过 `create()` 方法将补全和嵌入两套 `ModelClient` + `ResultConverter` 同时注册到 `Platform`，无需 API Key，仅通过 `$hostUrl` 定位本地 Docker Model Runner 服务。

## 关键方法

- `create(string $hostUrl, ?HttpClientInterface $httpClient, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform` — `$hostUrl` 默认为 `'http://localhost:12434'`，同时注册 `Completions\ModelClient`、`Embeddings\ModelClient`、`Embeddings\ResultConverter`、`Completions\ResultConverter`。

## 设计模式

- **无认证工厂**：本地服务无需 API Key，仅配置主机地址，降低使用门槛。

## 关联关系

- 同时注册补全和嵌入两组组件，支持混合模型目录。
- 不注册 Contract（传入 `null`），使用平台默认序列化逻辑。
