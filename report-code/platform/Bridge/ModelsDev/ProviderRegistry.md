# ProviderRegistry.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`ProviderRegistry` 在构造时加载所有供应商的元数据（名称和 API 基础 URL），提供供应商查询、API URL 获取、目录按需创建等功能，是 ModelsDev Bridge 中供应商信息的集中访问点。

## 关键方法

- `__construct(?string $dataPath)` — 通过 `DataLoader::load()` 初始化 `$providers` 映射（`providerId => {name, api}`）。
- `getApiBaseUrl(string $providerId): ?string` — 返回供应商的 API 基础 URL（去除尾部斜杠），供应商无 `api` 字段时返回 `null`。
- `getProviderName(string $providerId): string` — 返回供应商的显示名称。
- `getCatalog(string $providerId): ?ModelCatalog` — 按需创建并返回指定供应商的 `ModelCatalog` 实例，供应商不存在时返回 `null`。
- `has(string $providerId): bool` — 检查供应商是否存在于注册表中。
- `getProviderIds(): list<string>` — 返回所有供应商 ID 列表。

## 设计模式

- **延迟目录创建**：`getCatalog()` 每次调用创建新实例（依赖 `DataLoader` 缓存避免重复 I/O），避免预加载所有供应商目录的开销。

## 关联关系

- 依赖 `DataLoader`（数据来源）。
- 被 `PlatformFactory`（API URL 解析）和 `ModelResolver`（供应商查询）使用。
