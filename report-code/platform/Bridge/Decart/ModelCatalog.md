# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart`

## 概述

`ModelCatalog` 是 Decart 的静态模型目录，继承 `AbstractModelCatalog`，预定义 7 个 Lucy 系列多模态生成模型，覆盖文本到图像、文本到视频、图像/视频编辑等多种能力组合，支持通过 `$additionalModels` 注入自定义模型。

## 关键方法

- `__construct(array $additionalModels = [])` — 使用数组展开（`...$defaultModels, ...$additionalModels`）合并默认和自定义模型，初始化 `$this->models`。

## 设计模式

- **多能力模型定义**：同一模型可同时具备多个能力（如 `lucy-dev-i2v` 同时支持 `IMAGE_TO_VIDEO` 和 `VIDEO_TO_VIDEO`）。

## 关联关系

- 继承 `AbstractModelCatalog`，实现 `ModelCatalogInterface`。
- 所有条目的 `class` 字段均指向 `Decart::class`。
- 被 `PlatformFactory` 作为默认目录使用。
