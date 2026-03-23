# OpenResponses/ResultConverter 分析报告

解析 Responses API 的 `output[]` 数组，与 `OpenAi\Gpt\ResultConverter` 逻辑几乎相同（`OpenAi\Gpt\ResultConverter` 是此类的扩展版，添加了速率限制头解析）。处理 message/function_call/reasoning 三种输出类型。
