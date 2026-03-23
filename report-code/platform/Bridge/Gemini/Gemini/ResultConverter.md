# Gemini/ResultConverter.php 分析报告

## 文件概述

`Gemini\ResultConverter` 将 Gemini API 的原始 HTTP 响应解析为统一的平台结果类型，支持流式、工具调用、文本、图像、代码执行等多种响应模式。

## 关键方法分析

### `convert(RawResultInterface $result, array $options = []): ResultInterface`

主分发逻辑：
1. HTTP 429 → `RateLimitExceededException`
2. `$options['stream']` → `StreamResult`（包装 Generator）
3. 无 `candidates[0]` → 检查 `error` 字段抛出 `RuntimeException`
4. 多候选 → `ChoiceResult`；单候选 → 直接返回

### `convertChoice(array $choice): ToolCallResult|TextResult|BinaryResult|null`

按优先级解析单个候选：
1. 任意 part 含 `functionCall` → `ToolCallResult`（立即返回，忽略其余 part）
2. 单 part 含 `text` → `TextResult`
3. 单 part 含 `inlineData` → `BinaryResult::fromBase64()`（图像输出）
4. 多 part 场景：检测代码执行成功后提取后续文本
5. 其他 → 抛出异常（含 TODO 注释，issue #1053）

### `convertStream(RawResultInterface $result): \Generator`

从 SSE 数据流逐块提取 `getContent()`，对多候选块封装为 `ChoiceResult`。

## 设计特点

- **代码执行感知**：识别 `codeExecutionResult.outcome === OUTCOME_OK`，跳过执行块，仅提取后续输出文本
- `BinaryResult::fromBase64()` 处理 `inlineData`（图像/音频生成结果）
- `convertToolCall` 中 `$toolCall['id'] ?? ''` 容错处理（部分响应无 id）
- 流式模式下 Token 用量需调用方手动处理（`TokenUsageExtractor` 返回 null）

## 关联文件

- `TokenUsageExtractor.php` — 从非流式响应中提取 Token 用量
- `Gemini/ModelClient.php` — 提供原始 HTTP 响应
- `Symfony\AI\Platform\Result\*` — 所有结果类型
