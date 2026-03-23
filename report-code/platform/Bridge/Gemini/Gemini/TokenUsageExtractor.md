# Gemini/TokenUsageExtractor.php 分析报告

## 文件概述

从 Gemini API 的非流式响应中提取 Token 用量统计，包括提示词、输出、思考、工具和缓存等多维度令牌计数。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options = []): ?TokenUsageInterface`

```php
return new TokenUsage(
    promptTokens:     $content['usageMetadata']['promptTokenCount'] ?? null,
    completionTokens: $content['usageMetadata']['candidatesTokenCount'] ?? null,
    thinkingTokens:   $content['usageMetadata']['thoughtsTokenCount'] ?? null,
    toolTokens:       $content['usageMetadata']['toolUsePromptTokenCount'] ?? null,
    cachedTokens:     $content['usageMetadata']['cachedContentTokenCount'] ?? null,
    totalTokens:      $content['usageMetadata']['totalTokenCount'] ?? null,
);
```

**提前返回条件：**
- `$options['stream']` 为 true → 返回 null（流式令牌嵌入各 SSE chunk，需手动处理）
- 响应无 `usageMetadata` 键 → 返回 null

## 与 VertexAi 对应类的差异

| 字段 | Gemini | VertexAi |
|------|--------|---------|
| `toolUsePromptTokenCount` | ✅ 支持（`toolTokens`） | ❌ 不提取 |
| 实现结构 | 直接内联 | 抽取为 `extractUsageMetadata()` 私有方法 |

## 设计特点

- Gemini 2.5 系列支持 `thoughtsTokenCount`（思考令牌），此处通过 `thinkingTokens` 字段暴露
- 所有字段均为可选（`?? null`），响应缺少某字段时不报错
- 实现 `TokenUsageExtractorInterface`，由 `ResultConverter::getTokenUsageExtractor()` 返回

## 关联文件

- `Gemini/ResultConverter.php` — 调用 `getTokenUsageExtractor()` 返回此类实例
- `Symfony\AI\Platform\TokenUsage\TokenUsage` — 最终结果对象
