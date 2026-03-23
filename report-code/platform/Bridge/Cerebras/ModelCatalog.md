# ModelCatalog.php 分析报告

## 概述

`ModelCatalog` 静态定义了 Cerebras 平台当前支持的所有模型（包括 Llama、Qwen、GPT-OSS 等系列），所有模型均映射到 `Cerebras\Model` 类型，支持通过构造函数注入自定义模型扩展。

## 关键方法分析

### `__construct(array $additionalModels = [])`
定义 `$defaultModels` 数组，将每个模型 ID 映射到 `Model::class` 和对应能力列表，随后通过 `array_merge` 合并 `$additionalModels`。

## 预置模型与能力

所有模型均具备基础能力：`INPUT_MESSAGES`、`OUTPUT_STRUCTURED`、`OUTPUT_TEXT`、`OUTPUT_STREAMING`。  
以下模型额外支持 `TOOL_CALLING`：

| 模型 | 工具调用 |
|------|----------|
| `qwen-3-32b` | ✓ |
| `gpt-oss-120b` | ✓ |
| `zai-glm-4.7` | ✓ |

其余模型（`llama-4-scout-17b-16e-instruct`、`llama3.1-8b`、`llama-3.3-70b`、`llama-4-maverick-17b-128e-instruct`、`qwen-3-235b-a22b-instruct-2507`、`qwen-3-235b-a22b-thinking-2507`、`qwen-3-coder-480b`）不支持工具调用。

## 设计模式

- **继承 AbstractModelCatalog**：复用父类的 `getModel()`/`getModels()` 通用实现
- **扩展开放设计**：`$additionalModels` 参数允许追加自定义模型，满足测试或私有模型场景

## 与其他类的关系

- `PlatformFactory::create()` 默认使用此类
- 所有模型均使用 `Cerebras\Model` 类型，确保 `ModelClient` 和 `ResultConverter` 的 `supports()` 方法能正确匹配
