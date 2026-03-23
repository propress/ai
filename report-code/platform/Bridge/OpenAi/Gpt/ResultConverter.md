# OpenAi/Gpt/ResultConverter 分析报告

## 文件概述
解析 OpenAI Responses API 格式的响应，是 Bridge 中**最复杂的 ResultConverter**，处理 `output[]` 数组中的多种类型（message/function_call/reasoning），支持流式和非流式模式。

## 响应格式（Responses API）
```json
{
  "output": [
    {"type": "reasoning", "summary": [{"text": "..."}], "id": "..."},
    {"type": "message", "content": [{"type": "output_text", "text": "..."}]},
    {"type": "function_call", "name": "get_weather", "arguments": "{}", "id": "..."}
  ]
}
```
与旧的 `choices[].message.content` 格式完全不同。

## 方法分析

### `convertOutputArray(array $output): ResultInterface[]`
1. 提取所有 `function_call` 类型项 → `ToolCallResult`（可能有多个并行调用）
2. 处理剩余项（message/reasoning）→ `TextResult`
3. 多个结果 → `ChoiceResult`；单个结果 → 直接返回

### `convertOutputMessage(array $output): ?TextResult`
处理 `refusal` 类型（模型拒绝）：返回 `"Model refused to generate output: ..."` 字符串。

### `convertReasoning(array $item): ?ResultInterface`
提取推理摘要：`item['summary']['text']`（超出上下文限制时可能为 null）。

### 流式响应处理
流式 Responses API 使用 SSE 事件类型：
- `response.output_text.delta` → yield 文本片段
- `response.completed` → 提取 function_calls，yield `ToolCallResult`
- `response.usage` → yield `TokenUsage`（由 TokenUsage/StreamListener 消费）
- `error` → 抛出 `RuntimeException`

### 速率限制解析
`parseResetTime()` 解析 OpenAI 特有的重置时间格式：
- `"1s"` → 1
- `"6m0s"` → 360
- `"2m30s"` → 150

## 与其他文件的关系
- 使用 `Gpt\TokenUsageExtractor`
- 实现 `OpenResponses\ResultConverter` 的不同版本（独立实现，不继承）
