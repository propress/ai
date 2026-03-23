# ResultConverter.php 分析报告

## 概述

`ResultConverter` 将 Cerebras Chat Completions API 的 HTTP 响应转换为平台统一的结果类型，通过复用 `CompletionsConversionTrait` 支持流式和非流式响应处理，并对 Cerebras 特有的错误响应格式进行检测。

## 关键方法分析

### `supports(Model $model): bool`
通过 `$model instanceof Cerebras\Model` 确保只转换 Cerebras 模型的响应。

### `convert(RawResultInterface $result, array $options = []): ResultInterface`
处理两种响应路径：
1. **流式响应**：当 `$options['stream']` 为 true 时，调用 `CompletionsConversionTrait::convertStream()` 返回 `StreamResult`
2. **非流式响应**：解析 JSON 数据，检查 Cerebras 特有错误格式（`data['type']` 以 `error` 结尾），然后通过 `convertChoice()` 处理 `choices` 数组，单个选择直接返回，多个选择包装为 `ChoiceResult`

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，禁用 Token 用量统计。

## 设计模式

- **Trait 复用**：通过 `use CompletionsConversionTrait` 复用 Generic Bridge 的 `convertChoice()` 和 `convertStream()` 逻辑，避免重复实现
- **错误检测前置**：在解析 choices 之前先检查错误响应，提供清晰的错误信息

## 与其他类的关系

- 由 `PlatformFactory` 注册到 `Platform` 的 ResultConverter 列表
- 依赖 `Generic\Completions\CompletionsConversionTrait` 提供底层转换逻辑
- 消费 `ModelClient` 返回的 `RawHttpResult`，输出 `TextResult`、`ToolCallResult`、`ChoiceResult` 或 `StreamResult`
