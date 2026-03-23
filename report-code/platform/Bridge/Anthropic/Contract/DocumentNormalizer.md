# Anthropic/Contract/DocumentNormalizer 分析报告

将 `Document`（PDF 文件）序列化为 Anthropic document block（base64 格式）：`{type:'document', source:{type:'base64', media_type, data}}`。
