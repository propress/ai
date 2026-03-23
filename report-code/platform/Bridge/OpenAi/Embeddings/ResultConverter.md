# OpenAi/Embeddings/ResultConverter 分析报告

解析 OpenAI 嵌入响应，将 `data[].embedding` 转换为 `VectorResult`，提供详细的错误信息（包含 StatusCode 和响应体）以便调试。
