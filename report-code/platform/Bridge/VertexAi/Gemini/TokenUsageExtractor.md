# Gemini/TokenUsageExtractor.php 分析报告

## 文件概述

从 VertexAi Gemini API 的非流式响应中提取 Token 用量统计，与 Gemini Bridge 版本功能相同，但将提取逻辑抽取为私有方法，且不提取 `toolUsePromptTokenCount`。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options = []): ?TokenUsageInterface`

```php
// 流式场景直接返回 null
if ($options['stream'] ?? false) {
    return null;
}

// 无 usageMetadata 返回 null
if (!array_key_exists('usageMetadata', $content)) {
    return null;
}

return $this->extractUsageMetadata($content['usageMetadata']);
```

### `extractUsageMetadata(array $usage): TokenUsage`（私有方法）

```php
return new TokenUsage(
    promptTokens:     $usage['promptTokenCount'] ?? null,
    completionTokens: $usage['candidatesTokenCount'] ?? null,
    thinkingTokens:   $usage['thoughtsTokenCount'] ?? null,
    cachedTokens:     $usage['cachedContentTokenCount'] ?? null,
    totalTokens:      $usage['totalTokenCount'] ?? null,
);
```

## 与 Gemini Bridge 对应类的差异

| 字段 | Gemini | VertexAi |
|------|--------|---------|
| `toolUsePromptTokenCount` | ✅ 提取为 `toolTokens` | ❌ 不提取 |
| 实现结构 | 单方法内联 | 抽取 `extractUsageMetadata()` 私有方法 |
| PHPDoc 类型注解 | 无 | 有完整 array shape 注解 |

## 设计特点

- 使用 `array_key_exists` 检查键存在（而非 `isset`），能正确处理值为 null 的情况
- 私有方法拆分提高了可测试性和可读性
- VertexAi Gemini API 可能不返回 `toolUsePromptTokenCount`，因此此字段被省略

## 关联文件

- `Gemini/ResultConverter.php` — 调用 `getTokenUsageExtractor()` 返回此类实例
- `Symfony\AI\Platform\TokenUsage\TokenUsage` — 最终结果对象
