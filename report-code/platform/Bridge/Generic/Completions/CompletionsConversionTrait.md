# CompletionsConversionTrait 分析报告

## 文件概述
`CompletionsConversionTrait` 封装 OpenAI Chat Completions API 格式的**流式响应解析**和**非流式响应转换**逻辑，供 `Generic\Completions\ResultConverter` 和其他 Bridge 复用。

## Trait 方法分析

### `convertStream(RawResultInterface $result): \Generator`
流式 SSE 数据解析核心：
1. 遍历 `getDataStream()` 返回的 JSON 数据帧
2. 若是工具调用 delta：`convertStreamToToolCalls()` 累积增量参数
3. 工具调用完成（`finish_reason = 'tool_calls'`）：yield `ToolCallResult`
4. 普通文本 delta：yield 字符串片段

**关键技巧**：工具调用参数是增量传输的（每个帧只有 arguments 的一部分），必须跨帧累积才能得到完整 JSON。`$toolCalls[$i]['function']['arguments'] .= $toolCall['function']['arguments']` 实现了这个累积。

### `convertStreamToToolCalls(array $toolCalls, array $data): array`
增量工具调用累积：
- 有 `id` 字段 → 初始化新工具调用
- 无 `id` 字段 → 追加 arguments 到已有工具调用

### `convertChoice(array $choice): ToolCallResult|TextResult`
非流式响应的单个 choice 转换：
- `finish_reason = 'tool_calls'` → `ToolCallResult`
- `finish_reason = 'stop'|'length'` → `TextResult`
- 其他 → `RuntimeException`

### `convertToolCall(array $toolCall): ToolCall`
单个工具调用的 JSON 参数反序列化，空 arguments 返回 `[]`（而非 null）。

## 设计模式
**Trait 代码复用**：流式工具调用解析是复杂且易错的逻辑，通过 Trait 在 Generic 和 Ollama 等 Bridge 之间共享，避免各自维护副本。

## 与其他文件的关系
- 被 `Generic\Completions\ResultConverter` use
- 被部分兼容 OpenAI 格式流式 API 的 Bridge 复用
