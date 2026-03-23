# ModelCatalog.php 分析报告

## 概述

`ModelCatalog` 以静态配置方式定义了 Albert 平台的四个预置模型，继承自 `AbstractModelCatalog`，支持通过构造函数注入自定义模型进行扩展。

## 关键方法分析

### `__construct(array $additionalModels = [])`
- 预定义四个模型的类型映射与能力列表
- `$additionalModels` 参数允许调用方追加自定义模型，通过 `array_merge` 合并（自定义优先级更高，可覆盖同名默认模型）
- 将合并结果存入继承自父类的 `$models` 属性

## 预置模型能力

| 模型 | 类 | 能力 |
|------|----|------|
| `openweight-small` | `CompletionsModel` | `INPUT_MESSAGES`, `OUTPUT_TEXT`, `OUTPUT_STREAMING` |
| `openweight-medium` | `CompletionsModel` | `INPUT_MESSAGES`, `OUTPUT_TEXT`, `OUTPUT_STREAMING` |
| `openweight-large` | `CompletionsModel` | `INPUT_MESSAGES`, `OUTPUT_TEXT`, `OUTPUT_STREAMING` |
| `openweight-embeddings` | `EmbeddingsModel` | `INPUT_TEXT`, `EMBEDDINGS` |

## 设计模式

- **模板方法模式**：继承 `AbstractModelCatalog`，父类提供 `getModel()`/`getModels()` 的通用逻辑，子类只需填充 `$models` 数组
- **扩展点设计**：通过 `$additionalModels` 参数提供开放扩展、关闭修改的能力

## 与其他类的关系

- `PlatformFactory::create()` 默认使用此类作为模型目录，也可替换为其他 `ModelCatalogInterface` 实现
- 引用 `Generic\CompletionsModel` 和 `Generic\EmbeddingsModel` 作为模型类型标记，由 Generic Bridge 的 ModelClient 进行路由
