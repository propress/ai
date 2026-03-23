# Anthropic/Contract/ImageNormalizer 分析报告

将 `Image` 序列化为 Anthropic image block（base64 格式）：`{type:'image', source:{type:'base64', media_type, data}}`。注意 `jpg` → `jpeg` 的 MIME type 修正（Anthropic 不接受 `image/jpg`）。
