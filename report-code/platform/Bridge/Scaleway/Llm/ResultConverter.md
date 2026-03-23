# `Scaleway/Llm/ResultConverter.php` 分析

## 概述

实现 `ResultConverterInterface`，处理 Scaleway LLM 聊天补全响应的转换，通过 `CompletionsConversionTrait` 同时支持流式（SSE）和非流式两种输出格式。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Scaleway\Scaleway` 实例 |
| `convert(RawResultInterface, options): ResultInterface` | 流式时返回 `StreamResult`，非流式时提取 `choices` 返回 `ChoiceResult` 或单个 Choice |
| `getTokenUsageExtractor(): null` | 不提供 Token 用量提取 |

## 设计要点

- `use CompletionsConversionTrait` 引入 `convertStream()` 和 `convertChoice()` 方法，复用 OpenAI 兼容格式的解析逻辑
- 检测到 `error.code === 'content_filter'` 时抛出 `ContentFilterException`
- 多个 choices 时返回 `ChoiceResult`，单个 choice 时直接返回该 choice 对象

## 关系

- 实现：`ResultConverterInterface`
- 使用 Trait：`Bridge\Generic\Completions\CompletionsConversionTrait`
- 支持模型类型：`Scaleway\Scaleway`
- 被 `Scaleway\PlatformFactory` 注册
