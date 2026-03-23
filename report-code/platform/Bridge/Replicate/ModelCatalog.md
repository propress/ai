# `Replicate/ModelCatalog.php` 分析

## 概述

Replicate Bridge 的模型目录，继承 `AbstractModelCatalog`，预置 15 个 Meta Llama 系列模型，所有模型均仅支持 `INPUT_MESSAGES` 和 `OUTPUT_TEXT` 两种能力（无流式输出）。

## 关键方法

| 方法 | 说明 |
|------|------|
| `__construct(additionalModels)` | 构建默认 Llama 模型列表，与自定义模型合并后赋值给 `$this->models` |

## 设计要点

- 所有模型类统一为 `Bridge\Meta\Llama::class`
- 无 `OUTPUT_STREAMING` 能力，反映了 Replicate 轮询架构的固有限制
- 涵盖 Llama 3、3.1、3.2、3.3 各版本（含视觉模型 90B/11B）
- 支持通过 `additionalModels` 参数扩展，类型标注使用 `class-string`

## 关系

- 继承：`AbstractModelCatalog`
- 注册类型：全部为 `Bridge\Meta\Llama::class`
- 被 `PlatformFactory::create()` 作为默认目录参数使用
