# ModelCatalog.php 分析报告

## 文件概述

`ModelCatalog` 维护 Gemini Bridge 所有预定义模型的能力清单，支持通过构造参数注入自定义模型。

## 关键方法分析

### `__construct(array $additionalModels = [])`

- 在 `$defaultModels` 数组中硬编码所有内置模型，格式为 `模型名 => [class, capabilities]`
- 通过 `array_merge` 将用户自定义模型合并（自定义模型可覆盖默认值）

## 模型分类

| 类别 | 模型数量 | 类 | 说明 |
|------|---------|-----|------|
| **Gemini 3 系列** | 4 款 | `Gemini::class` | 含 pro/flash/image，支持 `THINKING` |
| **Gemini 2.5 系列** | 5 款 | `Gemini::class` | 主力生产模型，部分支持 `THINKING` |
| **Gemini 2.0 Flash** | 1 款 | `Gemini::class` | 标准生产模型 |
| **TTS 模型** | 3 款 | `Gemini::class` | 仅含 `TEXT_TO_SPEECH` + `OUTPUT_AUDIO` |
| **Embeddings** | 1 款 | `Embeddings::class` | `gemini-embedding-001` |

## 设计特点

- TTS 模型只声明 `INPUT_MESSAGES` + `OUTPUT_AUDIO` + `TEXT_TO_SPEECH`，不含文本输出能力
- `gemini-2.5-flash-lite-preview-09-2025` 与 `gemini-2.5-flash-lite` 能力相同，前者为历史预览版
- 图像生成模型（`gemini-3-pro-image-preview`、`gemini-2.5-flash-image`）含 `OUTPUT_IMAGE`
- 继承 `AbstractModelCatalog`，将数据存入受保护的 `$models` 属性供框架读取

## 关联文件

- `PlatformFactory.php` — 默认传入 `new ModelCatalog()` 实例
- `Gemini.php` / `Embeddings.php` — 被 `class` 字段引用，用于实例化模型对象
