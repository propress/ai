# ToolCallNormalizer 分析报告

## 文件概述
`ToolCallNormalizer` 将 `Result\ToolCall` 对象（模型返回的工具调用请求）序列化为 OpenAI 标准格式，用于将工具调用历史回传给模型（作为 AssistantMessage 的 tool_calls 字段）。

## 输出格式
```json
{
    "id": "call_abc123",
    "type": "function",
    "function": {
        "name": "get_weather",
        "arguments": "{\"city\": \"Beijing\"}"
    }
}
```
注意：`arguments` 是 JSON 字符串而非对象（OpenAI API 要求）。

## 技巧
空参数使用 `json_encode(new \stdClass())` → `"{}"` 而非 `json_encode([])` → `"[]"`，确保空参数序列化为对象而非数组。

## 与其他文件的关系
- 序列化 `Result/ToolCall.php` 对象
- 被 `AssistantMessageNormalizer` 通过委托调用
