# `Replicate/LlamaResultConverter.php` 分析

## 概述

实现 `ResultConverterInterface`，将 Replicate 响应中的 `output` 字段（字符串数组）拼接为完整文本，封装为 `TextResult` 返回。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Meta\Llama` 实例 |
| `convert(RawResultInterface, options): TextResult` | 取 `data['output']`（字符串数组），用 `implode('', ...)` 拼接后返回 `TextResult` |
| `getTokenUsageExtractor(): null` | 不提供 Token 用量提取 |

## 设计要点

- Replicate 的 `output` 字段是逐 token 生成的字符串数组（非单一字符串），需要 `implode` 拼接
- 缺少 `output` 键时抛出 `RuntimeException`
- 不支持流式输出（Replicate 使用轮询而非 SSE）

## 关系

- 实现：`ResultConverterInterface`
- 支持模型类型：`Bridge\Meta\Llama`
- 输出类型：`TextResult`
- 被 `PlatformFactory` 注册
