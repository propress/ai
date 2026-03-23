# Anthropic/Contract/ToolNormalizer 分析报告

将 `Tool` 序列化为 Anthropic 格式：`{name, description, input_schema}`（与 OpenAI 的 `{type:'function', function:{name, description, parameters}}` 不同）。空参数时 `input_schema` 为 `{type:'object'}`。
