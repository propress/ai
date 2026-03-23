# Toolbox/StreamListener.php 分析报告

## 概述

`StreamListener` 继承自 `AbstractStreamListener`（Platform 模块），在流式 AI 响应中监听工具调用块，当检测到 `ToolCallResult` 时调用 `handleToolCallsCallback` 回调执行工具并将后续结果写入流，实现流式响应中的工具调用支持。

## 关键方法分析

### onStart(StartEvent $event): void

重置状态：清空 `$buffer`，将 `$toolHandled` 标志置为 `false`。

### onChunk(ChunkEvent $event): void

核心处理逻辑：

1. 若 `$toolHandled` 为 `true`（本流中已处理过工具调用），跳过所有后续块（`skipChunk()`）。
2. 若当前块是字符串，追加到 `$buffer`（累积流式文本响应，用于构建 `AssistantMessage`）。
3. 若当前块是 `ToolCallResult`：
   - 调用 `($this->handleToolCallsCallback)($chunk, Message::ofAssistant($this->buffer))`，传入已累积的 AssistantMessage。
   - 将回调返回的最终结果的内容设为本块的新值（`setChunk()`）。
   - 标记 `$toolHandled = true`，后续块全部跳过。

### onComplete(CompleteEvent $event): void

重置状态，并将工具调用后最终结果的元数据合并到流完成事件的元数据中（如 `sources`）。

## 设计模式

- **模板方法（Template Method）**：继承 `AbstractStreamListener`，实现 `onStart`、`onChunk`、`onComplete` 三个模板钩子方法。
- **状态机（State Machine）**：`$toolHandled` 标志将监听器的状态从「正常流式」切换到「工具已处理，跳过后续块」。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentProcessor` | 创建 `StreamListener` 实例，传入 `handleToolCallsCallback` 闭包 |
| `Platform/Result/Stream/AbstractStreamListener` | 父类，定义流事件接口 |
| `Platform/Result/ToolCallResult` | 在 `onChunk()` 中检测并处理 |
