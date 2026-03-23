# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner`

## 概述

`ModelCatalog` 是 DockerModelRunner 的静态模型目录，继承 `AbstractModelCatalog`，预定义 16 个补全模型（`Completions` 类型）和 4 个嵌入模型（`Embeddings` 类型），均以 `ai/` 前缀命名，支持通过 `$additionalModels` 注入自定义模型。

## 关键方法

- `__construct(array $additionalModels = [])` — 通过 `array_merge($defaultModels, $additionalModels)` 合并模型，初始化 `$this->models`。

## 设计模式

- **双类型目录**：在同一目录中混合 `Completions` 和 `Embeddings` 两种模型类，通过 `class` 字段区分，平台根据此字段路由到对应的 `ModelClient` 和 `ResultConverter`。

## 关联关系

- 继承 `AbstractModelCatalog`，实现 `ModelCatalogInterface`。
- 被 `PlatformFactory` 作为默认目录使用。
