# ModelCatalog.php 分析报告

## 概述

`ModelCatalog` 是 OpenRouter Bridge 的静态模型目录，通过手动维护的方式定义了大量 OpenRouter 平台上可用的模型（涵盖来自 x-ai、Google、Anthropic、OpenAI、Meta 等厂商的主流模型），适用于无需动态 API 调用的生产环境场景。

## 关键方法分析

### `__construct(array $additionalModels = [])`
调用父类 `AbstractOpenRouterModelCatalog` 构造函数初始化路由器条目，随后追加大量静态定义的模型，并与 `$additionalModels` 合并，允许调用方扩展自定义模型。

### `getModel(string $modelName): Model`
重写父类方法，提供对模型名称的宽松匹配支持（通过名称后缀如 `:free`、`:nitro`、`:thinking` 等变体映射），使得 OpenRouter 的模型变体语法能被正确解析。

## 静态模型来源

模型列表在 `2025-11-21` 生成，注释明确标注此为时间快照，建议使用 `ModelApiCatalog` 获取最新完整列表。包含以下来源的模型：
- **x-ai**：Grok 系列
- **Google**：Gemini 系列
- **Anthropic**：Claude 系列（通过 OpenRouter 路由）
- **Meta**：Llama 系列
- **OpenAI**：GPT/o 系列
- **其他**：Mistral、DeepSeek 等多个厂商

## 设计模式

- **继承模板方法**：继承 `AbstractOpenRouterModelCatalog`，自动获得路由器条目和 `@preset` 解析
- **时间快照策略**：以代码注释方式标注生成时间，提醒维护者定期更新
- **扩展开放**：通过 `$additionalModels` 参数支持自定义模型追加

## 与其他类的关系

- `PlatformFactory::create()` 默认使用此类，因其无需 HTTP 请求即可工作
- 可被 `ModelApiCatalog` 替代，获得实时、完整的模型列表
