# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`ModelCatalog` 是 ElevenLabs 的静态模型目录，继承 `AbstractModelCatalog`，预定义 10 个 TTS 模型（eleven_v3 系列、turbo 系列等）和 2 个 STT 模型（scribe_v1 系列），并支持通过 `$additionalModels` 注入自定义模型。

## 关键方法

- `__construct(array $additionalModels = [])` — 合并默认模型定义与附加模型，初始化 `$this->models`。

## 设计模式

- **静态预定义目录**：与 `ElevenLabsApiCatalog` 形成互补，适合离线或不需要实时模型列表的场景。

## 关联关系

- 继承 `AbstractModelCatalog`，实现 `ModelCatalogInterface`。
- 被 `PlatformFactory` 在 `$apiCatalog=false`（默认）时使用。
- 所有条目的 `class` 字段均指向 `ElevenLabs::class`。
