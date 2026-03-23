# OpenResponses/Contract/ToolCallNormalizer 分析报告

将 `ToolCall` 序列化为 Responses API 格式，用于多轮工具调用时将历史工具调用回传：`{type:'function_call', call_id, name, arguments}`。空参数序列化为 `{}` 而非 `[]`（确保 JSON object 格式）。
