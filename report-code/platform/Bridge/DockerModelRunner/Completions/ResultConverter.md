# Completions/ResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner\Completions`

## 概述

`ResultConverter` 实现 `ResultConverterInterface`，通过 `use CompletionsConversionTrait` 复用 Generic Bridge 的补全响应转换逻辑，并额外处理 Docker Model Runner 特有的错误码（模型未找到、超出上下文大小、内容过滤）。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Completions` 实例。
- `convert(RawResultInterface $result, array $options): ResultInterface` — 流式时返回 `StreamResult`；非流式时依次检查 404/模型未找到、`exceed_context_size_error`、`content_filter` 错误，再提取 `choices` 构建 `ChoiceResult`。
- `getTokenUsageExtractor(): ?TokenUsageExtractorInterface` — 返回 `null`。

## 设计模式

- **Trait 复用**：`CompletionsConversionTrait` 提供流解析和 choice 转换逻辑，避免重复实现。

## 关联关系

- 使用 `CompletionsConversionTrait`（来自 `Bridge\Generic\Completions`）。
- 抛出 `ModelNotFoundException`、`ExceedContextSizeException`、`ContentFilterException`（均为平台级专用异常）。
