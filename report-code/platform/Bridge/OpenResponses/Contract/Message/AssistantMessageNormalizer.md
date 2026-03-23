# OpenResponses/Contract/Message/AssistantMessageNormalizer 分析报告

将 `AssistantMessage` 序列化为 Responses API 格式，工具调用时委托给 `ToolCallNormalizer`（返回 function_call 数组），普通文本返回 `{role:'assistant', type:'message', content:...}`。
