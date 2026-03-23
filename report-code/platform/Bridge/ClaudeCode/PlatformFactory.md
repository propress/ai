# PlatformFactory.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`PlatformFactory` 是 ClaudeCode 桥接器的静态工厂类，通过 `create()` 方法将 `ModelClient`、`ResultConverter`、`ModelCatalog` 和 `ClaudeCodeContract` 组装为完整的 `Platform` 实例，是对外的唯一入口。

## 关键方法

- `create(?string $cliBinary, ?string $workingDirectory, ?float $timeout, array $environment, ModelCatalogInterface $modelCatalog, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher, LoggerInterface $logger): Platform` — 创建并返回配置好的 `Platform` 实例。

## 设计模式

- **静态工厂方法**：所有参数均有合理默认值（timeout=300s），最简可直接 `PlatformFactory::create()` 调用。

## 关联关系

- 组装 `ModelClient`、`ResultConverter`、`ModelCatalog`（默认）、`ClaudeCodeContract`（默认）。
- 最终创建并返回 `Platform` 实例。
