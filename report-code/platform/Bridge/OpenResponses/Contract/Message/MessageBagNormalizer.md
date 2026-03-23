# OpenResponses/Contract/Message/MessageBagNormalizer 分析报告

将 `MessageBag` 序列化为 Responses API 格式：`{input:[], instructions?:}`。AssistantMessage 含工具调用时用 `array_merge` 展开（多个 function_call 条目），而非 `[]=` 推入（单个 message 条目）。
