# ChunkEvent 分析报告

## 文件概述
`ChunkEvent` 携带单个流式数据块（chunk），是三种流事件中唯一携带数据的事件。监听器可以读取、修改或跳过 chunk。

## 类定义
- **类型**: `final class`，继承 `Event`

## 方法分析

### `__construct(StreamResult $result, mixed $chunk)`
- `$chunk`：原始数据块（可以是字符串片段、`ThinkingContent`、`ToolCall` 部分、`TokenUsage` 等）

### `setChunk(mixed $chunk): void` / `getChunk(): mixed`
- 允许监听器修改 chunk 内容（如过滤、转换格式）

### `skipChunk(): void` / `isChunkSkipped(): bool`
- 标记跳过此 chunk（不追加到输出文本）
- **典型用途**：`TokenUsage\StreamListener` 收到 `TokenUsageInterface` chunk 后调用 `skipChunk()`，将使用量存入 Metadata 而非追加到文本中

## 设计模式
**命令（Command）+ 可观察事件**：chunk 的跳过/修改由监听器决定，`StreamResult` 的迭代逻辑只需检查 `isChunkSkipped()`，实现数据流处理的插件化。

## 与其他文件的关系
- 被 `AbstractStreamListener::onChunk()` 接收
- 被 `TokenUsage\StreamListener` 用于提取并跳过使用量块
