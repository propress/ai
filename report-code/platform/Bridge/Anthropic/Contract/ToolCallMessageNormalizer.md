# Anthropic/Contract/ToolCallMessageNormalizer 分析报告

将工具调用结果（`ToolCallMessage`）转换为 Anthropic 格式：
```json
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "...", "content": "..."}]}
```
与 OpenAI 格式（`role: 'tool', tool_call_id`）完全不同：Anthropic 将工具结果作为 user 消息的 content 块。
