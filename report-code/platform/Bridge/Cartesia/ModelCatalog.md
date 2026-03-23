# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia`

## 概述

`ModelCatalog` 是 Cartesia 的静态模型目录，继承 `AbstractModelCatalog`，预定义 2 个模型：TTS 模型 `sonic-3` 和 STT 模型 `ink-whisper`，支持通过 `$additionalModels` 注入自定义模型。

## 关键方法

- `__construct(array $additionalModels = [])` — 使用数组展开（`...$defaultModels, ...$additionalModels`）合并默认和自定义模型，初始化 `$this->models`。

## 设计模式

- **最小化静态目录**：仅定义当前主力模型，保持目录简洁，用户可按需扩展。

## 关联关系

- 继承 `AbstractModelCatalog`，实现 `ModelCatalogInterface`。
- 被 `PlatformFactory` 作为默认目录使用。
