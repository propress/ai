# DataLoader.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`DataLoader` 是一个带静态缓存的 JSON 文件加载器（`@internal`），负责读取并缓存 models.dev 的 JSON 数据文件，同时对供应商按字母排序、对模型按发布日期降序排列，避免重复文件 I/O 操作。

## 关键方法

- `load(?string $dataPath): array` — 优先返回缓存数据（路径相同时），否则读取文件、解析 JSON 并排序后缓存。默认路径通过 `resolveDefaultPath()` 从 `symfony/models-dev` Composer 包解析。
- `clearCache(): void` — 清除静态缓存，仅供测试使用。
- `resolveDefaultPath(): string` — 通过 `Composer\InstalledVersions` 查找 `symfony/models-dev` 包的安装路径，未安装时抛出 `RuntimeException`。
- `sortData(array $data): array` — 供应商 `ksort()` 字母排序 + 模型 `uasort()` 按 `release_date` 降序。

## 设计模式

- **类级静态缓存**：`$cachedData` 和 `$cachedPath` 确保相同路径的 JSON 只加载一次，适合同一请求中多次构建目录的场景。

## 关联关系

- 被 `ModelCatalog`、`ProviderRegistry`、`PlatformFactory` 调用。
- 依赖 `Composer\InstalledVersions` 进行默认路径解析。
