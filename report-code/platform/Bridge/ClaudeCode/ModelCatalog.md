# ModelCatalog.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`ModelCatalog` 是 ClaudeCode 的静态模型目录，继承 `AbstractModelCatalog`，在构造时预定义 `opus`、`sonnet`、`haiku` 三个模型，并支持通过 `$additionalModels` 参数注入自定义模型。

## 关键方法

- `__construct(array $additionalModels = [])`: 合并默认模型定义与附加模型，初始化 `$this->models`。

## 设计模式

- **数组合并扩展**：通过 `array_merge($defaultModels, $additionalModels)` 允许用户在不修改源码的情况下扩展模型目录。

## 关联关系

- 继承 `AbstractModelCatalog`（实现 `ModelCatalogInterface`）。
- 被 `PlatformFactory` 作为默认目录使用。
- 每个模型条目的 `class` 字段均指向 `ClaudeCode::class`。
