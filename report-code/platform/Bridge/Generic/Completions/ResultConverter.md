# Generic/Completions/ResultConverter 分析报告

## 文件概述
`Generic\Completions\ResultConverter` 解析 OpenAI Chat Completions API 的响应，将 HTTP 响应转换为 `TextResult`、`ToolCallResult`、`ChoiceResult` 或 `StreamResult`，覆盖所有正常和错误情况。

## 方法分析

### `convert(RawResultInterface $result, array $options): ResultInterface`
1. HTTP 状态码检查：401/400/429 → 精确异常
2. 流式模式：`return new StreamResult($this->convertStream($result))`
3. 非流式：解析 `choices[]`
4. 单个 choice → 直接返回；多个 choice → `ChoiceResult` 包装

### 错误处理覆盖范围
- 401 `AuthenticationException`（附错误消息）
- 400 `BadRequestException`
- 429 `RateLimitExceededException`
- `content_filter` → `ContentFilterException`
- 无 `choices` → `RuntimeException`

## 与其他文件的关系
- use `CompletionsConversionTrait`（流式和工具调用解析）
- `getTokenUsageExtractor()` 返回 `new TokenUsageExtractor()`
