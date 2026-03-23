# CapabilityMapper.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`CapabilityMapper` 是一个纯静态工具类，将 models.dev JSON 中模型的元数据字段（如 `tool_call`、`reasoning`、`modalities` 等）映射为 Symfony AI Platform 的 `Capability` 枚举列表，同时提供嵌入模型检测逻辑。

## 关键方法

- `map(array $modelData): list<Capability>` — 入口方法，先判断是否为嵌入模型，分别调用 `mapEmbeddingCapabilities()` 或 `mapCompletionCapabilities()`。
- `isEmbeddingModel(array $modelData): bool` — 检查 `family` 字段或 `id` 中是否含 `embed` 关键字。
- `mapCompletionCapabilities(array $modelData): list<Capability>` — 基础能力包含 `INPUT_MESSAGES`、`OUTPUT_TEXT`、`OUTPUT_STREAMING`，根据 `tool_call`、`structured_output`、`reasoning` 及输入/输出 modalities 动态追加。
- `mapEmbeddingCapabilities(array $modelData): list<Capability>` — 固定返回 `[INPUT_TEXT, EMBEDDINGS]`。

## 设计模式

- **元数据驱动能力推断**：将 AI 供应商的 JSON 描述字段翻译为平台抽象能力，是 ModelsDev Bridge 动态性的核心。

## 关联关系

- 被 `ModelCatalog` 在构建模型条目时调用。
