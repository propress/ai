# Anthropic/ResultConverter 分析报告

## 文件概述
解析 Anthropic Messages API 响应，处理文本、工具调用和 Extended Thinking 内容，支持流式（SSE）和非流式两种模式。

## 非流式响应处理
```
data['content'][].type == 'tool_use' → ToolCall
data['content'][].type == 'text'     → TextResult（text 字段）
```
**注意**：文本和工具调用可以共存，但返回时以工具调用优先（若有 toolCalls 则返回 `ToolCallResult` 而非 `TextResult`）。

## 流式响应处理（convertStream）
Anthropic SSE 事件类型映射：
| 事件类型 | 处理逻辑 |
|---|---|
| `content_block_start` + `type='thinking'` | 初始化 thinking 累积变量 |
| `content_block_delta` + `thinking_delta` | 累积思维链文本 |
| `content_block_delta` + `signature_delta` | 累积签名 |
| `content_block_stop`（thinking 完成）| yield `ThinkingContent(thinking, signature)` |
| `content_block_start` + `type='tool_use'` | 初始化工具调用（记录 id/name） |
| `content_block_delta` + `input_json_delta` | 累积工具调用 JSON arguments |
| `content_block_stop`（tool_use 完成）| 反序列化 JSON → `ToolCall` 加入列表 |
| `content_block_delta` + `text` | yield 文本片段 |
| `message_stop` + toolCalls 非空 | yield `ToolCallResult` |

**关键技巧**：思维链和工具调用参数都是增量传输的，需要跨事件帧累积。使用状态变量（`$currentThinking`, `$currentToolCall`, `$currentToolCallJson`）追踪当前未完成的块。

## 与其他文件的关系
- `getTokenUsageExtractor()` 返回 `TokenUsageExtractor`（Anthropic 专有字段）
- 流式 `ThinkingContent` 由 Agent 模块的 `StreamingResponse` 收集后存入 `AssistantMessage`
