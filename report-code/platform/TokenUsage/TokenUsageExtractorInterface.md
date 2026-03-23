# TokenUsageExtractorInterface 分析报告

## 文件概述
`TokenUsageExtractorInterface` 定义从 AI 原始响应中提取 Token 使用量的策略接口，每个 Bridge 实现自己的提取逻辑。

## 接口定义
```php
interface TokenUsageExtractorInterface {
    public function extract(RawResultInterface $rawResult, array $options = []): ?TokenUsageInterface;
}
```

## 方法分析

### `extract(RawResultInterface $rawResult, array $options): ?TokenUsageInterface`
- 从原始 HTTP 响应的 JSON 数据中解析 Token 使用量
- 返回 `null` 表示响应中不包含使用量信息
- `$options` 可传递模型名等上下文（如流式模式下选项不同）

## 设计模式
**策略（Strategy）**：不同 AI 提供商的 Token 使用量字段名、结构各不相同（如 Anthropic 的 `input_tokens`/`output_tokens` vs OpenAI 的 `prompt_tokens`/`completion_tokens`），通过接口允许 Bridge 各自实现。

## 与其他文件的关系
- 被各 Bridge 的 `ResultConverter` 实现
- 被 `ResultConverterInterface::getTokenUsageExtractor()` 暴露
