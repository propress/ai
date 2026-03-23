# ElevenLabsApiCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`ElevenLabsApiCatalog` 实现 `ModelCatalogInterface`，通过 `GET /models` 接口从 ElevenLabs API 动态获取所有可用模型及其能力，是静态 `ModelCatalog` 的运行时替代方案，能始终反映 API 最新的模型列表。

## 关键方法

- `getModel(string $modelName): ElevenLabs` — 先调用 `getModels()` 获取完整列表，若模型不存在或能力为空则抛出 `InvalidArgumentException`，否则返回对应的 `ElevenLabs` 模型实例。
- `getModels(): array` — 调用 `GET models`，将响应中每个模型的 `can_do_text_to_speech` / `can_do_voice_conversion` 字段映射为对应 `Capability` 数组。

## 设计模式

- **动态目录**：与静态 `ModelCatalog` 形成互补，适合需要实时感知新模型的场景。
- **能力推断**：通过 API 响应的布尔字段（而非硬编码）推断模型能力。

## 关联关系

- 实现 `ModelCatalogInterface`。
- 由 `PlatformFactory::create($apiCatalog=true)` 选用。
- 依赖 `HttpClientInterface` 发起 API 请求。
