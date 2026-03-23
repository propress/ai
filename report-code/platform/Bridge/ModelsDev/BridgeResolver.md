# BridgeResolver.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`BridgeResolver` 是一个纯静态工具类，维护 `BRIDGE_MAPPING` 常量，将 models.dev 中供应商的 NPM 包标识（如 `@ai-sdk/anthropic`）映射到对应的 Symfony 专用桥接器工厂类、Composer 包名、可路由性标志及模型类名，供 `PlatformFactory` 判断是否需要绕过通用 Generic Bridge。

## 关键方法

- `requiresSpecializedBridge(string $npmPackage): bool` — 检查 NPM 包是否在映射表中。
- `getBridgeFactory(string $npmPackage): ?string` — 返回对应的工厂类名（class-string）。
- `getComposerPackage(string $npmPackage): ?string` — 返回需要安装的 Composer 包名。
- `isBridgeAvailable(string $npmPackage): bool` — 通过 `class_exists()` 检查工厂类是否已安装。
- `isRoutable(string $npmPackage): bool` — 检查工厂签名是否兼容自动路由（`routable` 字段）。
- `getCompletionsModelClass()` / `getEmbeddingsModelClass()` — 返回供应商专用的模型类名（如 `Claude::class`）。

## 设计模式

- **静态映射表**：通过常量数组集中管理供应商到桥接器的映射，新增供应商只需扩展 `BRIDGE_MAPPING`。

## 关联关系

- 被 `PlatformFactory` 在创建 Platform 前查询，决定使用专用还是通用桥接器。
