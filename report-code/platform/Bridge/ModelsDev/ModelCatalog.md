# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`ModelCatalog` 继承 `AbstractModelCatalog`，基于 models.dev JSON 数据为指定供应商（`$providerId`）动态构建模型目录，自动过滤废弃模型，通过 `CapabilityMapper` 推断能力，默认使用 `CompletionsModel` / `EmbeddingsModel` 作为模型类，均可通过参数覆盖。

## 关键方法

- `__construct(string $providerId, ?string $dataPath, array $additionalModels, ?string $completionsModelClass, ?string $embeddingsModelClass)` — 加载数据、过滤 `deprecated` 模型，为每个模型映射能力并构建条目，最终与 `$additionalModels` 合并。

## 设计模式

- **供应商参数化**：同一类通过不同 `$providerId` 参数复用于所有供应商，无需为每个供应商创建独立目录类。
- **模型类可替换**：`$completionsModelClass` / `$embeddingsModelClass` 参数支持专用桥接器（如 Anthropic）使用自己的模型类。

## 关联关系

- 依赖 `DataLoader`（数据加载）和 `CapabilityMapper`（能力推断）。
- 被 `PlatformFactory`、`ProviderRegistry`、`ModelResolver` 使用。
