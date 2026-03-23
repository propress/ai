# ModelCatalog.php 分析报告

## 概述

`ModelCatalog` 静态定义了 Meta Llama API 提供的全系列 Llama 模型，所有模型均映射到 `Llama` 类型，部分视觉模型（Vision Instruct）额外支持图像输入能力，注意所有模型均不包含流式输出能力（`OUTPUT_STREAMING`）。

## 关键方法分析

### `__construct(array $additionalModels = [])`
定义包含 15+ 个 Llama 模型的 `$defaultModels` 数组，通过 `array_merge` 与 `$additionalModels` 合并，结果存入 `$models`。

## 模型能力分类

**仅文本输出模型**（`INPUT_MESSAGES` + `OUTPUT_TEXT`）：
- `llama-3.3-70B-Instruct`、`llama-3.2-3b`、`llama-3.2-3b-instruct`
- `llama-3.2-1b`、`llama-3.2-1b-instruct`
- `llama-3.1-405b-instruct`、`llama-3.1-70b`、`llama-3.1-8b`、`llama-3.1-8b-instruct`
- `llama-3-70b-instruct`、`llama-3-70b`、`llama-3-8b-instruct`、`llama-3-8b`

**视觉模型**（`INPUT_MESSAGES` + `OUTPUT_TEXT` + `INPUT_IMAGE`）：
- `llama-3.2-90b-vision-instruct`
- `llama-3.2-11b-vision-instruct`

> 所有模型均**不支持**流式输出（无 `OUTPUT_STREAMING`），这与其他大多数 Bridge 不同。

## 设计模式

- **继承 AbstractModelCatalog**：复用父类的通用模型查询逻辑
- **扩展开放**：`$additionalModels` 参数允许追加私有部署或测试模型

## 与其他类的关系

- 所有模型使用 `Llama::class` 类型，配合 `MessageBagNormalizer::supportsModel()` 的 `instanceof Llama` 检查
- Meta Bridge 无 `PlatformFactory`，此 Catalog 需手动传入 Platform 构造
