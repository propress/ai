# OpenAi/Embeddings/ModelClient 分析报告

向 `/v1/embeddings` 发送 POST 请求，请求格式：`{model, input}`，支持单文本或字符串数组批量嵌入。继承 `AbstractModelClient`（API Key 验证 + 区域选择）。
