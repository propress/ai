# OllamaMessageChunk.php 分析

## 概述

`OllamaMessageChunk` 是 Ollama 流式响应的单个块值对象，实现 `Stringable` 接口，封装 Ollama `/api/chat` 流式 JSON 行的完整字段，并提供对 `content`、`thinking`、`role` 和 `done` 的语义访问器。

## 关键方法分析

### `__toString(): string`
返回 `$this->message['content'] ?? ''`，使流块可以直接被字符串化为助手的回复文本，便于简单消费场景。

### `getContent(): ?string`
返回当前块中助手消息的文本内容（大多数块仅包含增量文本）。

### `getThinking(): ?string`
返回当前块中的思考内容（`message.thinking` 字段），用于支持 `thinking` 能力的模型（如 QwQ、DeepSeek-R1 等 Ollama 格式模型）。

### `getRole(): ?string`
返回消息角色（`message.role`），通常为 `"assistant"`。

### `isDone(): bool`
返回 `$this->done`，标识这是否是流的最后一个块。最后一个块通常不含内容，但包含性能统计信息（`eval_count` 等）。

## 关键模式

- **`Stringable` 实现**：允许将流块对象直接用于字符串上下文（如 `echo`、字符串拼接），简化普通文本流消费。
- **`$raw` 字段**：保存完整的原始 JSON 解析数组，供需要访问 Ollama 特有字段（如 `eval_count`、`eval_duration`）的消费方使用。
- **专用于 Ollama 协议**：与 Platform 层的通用 SSE 流块不同，此类专门封装 Ollama 的换行 JSON 流格式。

## 关联关系

- 由 `OllamaResultConverter::convertStream()` 为每个流块实例化并 yield。
- 消费方可通过 `getThinking()` 区分"思考过程"和"最终回答"两类内容流。
