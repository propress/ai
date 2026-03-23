# `Azure/Meta/LlamaResultConverter.php` 分析

## 概述

实现 `ResultConverterInterface`，将 Azure Llama 端点返回的原始 HTTP 响应转换为 `TextResult`，提取 `choices[0].message.content` 字段的文本内容。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Meta\Llama` 实例 |
| `convert(RawResultInterface, options): TextResult` | 从响应数据中取 `choices[0]['message']['content']`，不存在则抛出 `RuntimeException` |
| `getTokenUsageExtractor(): null` | 不提供 Token 用量提取器 |

## 设计要点

- 输出类型固定为 `TextResult`（不支持流式）
- 使用项目专用异常 `Platform\Exception\RuntimeException`

## 关系

- 实现：`ResultConverterInterface`
- 支持模型类型：`Bridge\Meta\Llama`
- 被 `Azure\Meta\PlatformFactory` 注册
