# OpenResponses/Contract/Message/ToolCallMessageNormalizer 分析报告

工具调用结果 → Responses API 格式：`{type:'function_call_output', call_id, output}`，使用 `call_id` 而非 OpenAI Chat Completions 的 `tool_call_id`。
